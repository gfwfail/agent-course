# 216. Agent Artifact Store 与文件生命周期管理

> 关键词：Artifact Store、临时文件、引用传递、自动清理、文件权限、跨工具交付

前面我们讲过多模态输出、富媒体响应、工具结果分页。今天讲一个更工程化、但生产里非常关键的问题：**Agent 生成的文件到底该放哪、怎么传、什么时候删？**

很多 Agent Demo 里会直接把文件路径塞给 LLM：

```text
/var/folders/tmp/report-final-v2.xlsx
```

这在本机 demo 能跑，但一到生产就会炸：

- Web / Telegram / Discord 需要的是可下载附件，不是本地路径
- Sub-agent 可能在另一台机器，读不到这个路径
- 文件可能包含敏感数据，不能永久公开
- 大文件不应该直接塞进上下文
- 临时文件没人清理，磁盘迟早爆

所以成熟 Agent 需要一个 **Artifact Store（产物存储）**。

---

## 1. Artifact 不是普通文件

Artifact 是 Agent 工作流中产生、引用、交付的文件型结果，比如：

- 报表：`billing-report.xlsx`
- 图片：`design-preview.png`
- PDF：`invoice.pdf`
- 日志包：`debug-bundle.tar.gz`
- 代码补丁：`fix.patch`
- 音频：`story.mp3`

它和普通文件的区别在于：Artifact 要有元数据。

```ts
export type Artifact = {
  id: string;              // art_01HX...
  name: string;            // report.xlsx
  mimeType: string;        // application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
  sizeBytes: number;
  storageKey: string;      // artifacts/2026/05/01/art_01HX.xlsx
  visibility: "private" | "signed-url" | "public";
  ownerSessionId: string;
  createdAt: string;
  expiresAt?: string;
  sha256: string;
  meta?: Record<string, unknown>;
};
```

**核心思想：LLM 不直接管理文件路径，只管理 artifact id。**

---

## 2. learn-claude-code 教学版：本地 Artifact Store

教学实现可以先用本地目录 + JSON 索引。

```python
# learn-claude-code style: simple local artifact store
from dataclasses import dataclass, asdict
from pathlib import Path
import hashlib, json, shutil, time, uuid

@dataclass
class Artifact:
    id: str
    name: str
    mime_type: str
    size_bytes: int
    path: str
    created_at: float
    expires_at: float | None
    sha256: str

class LocalArtifactStore:
    def __init__(self, root: str = ".agent_artifacts"):
        self.root = Path(root)
        self.root.mkdir(parents=True, exist_ok=True)
        self.index_path = self.root / "index.json"
        self.index = self._load_index()

    def _load_index(self):
        if self.index_path.exists():
            return json.loads(self.index_path.read_text())
        return {}

    def _save_index(self):
        self.index_path.write_text(json.dumps(self.index, indent=2, ensure_ascii=False))

    def put_file(self, src: str, name: str, mime_type: str, ttl_seconds: int | None = 86400):
        artifact_id = "art_" + uuid.uuid4().hex[:16]
        src_path = Path(src)
        dest = self.root / f"{artifact_id}_{name}"
        shutil.copyfile(src_path, dest)

        data = dest.read_bytes()
        artifact = Artifact(
            id=artifact_id,
            name=name,
            mime_type=mime_type,
            size_bytes=len(data),
            path=str(dest),
            created_at=time.time(),
            expires_at=time.time() + ttl_seconds if ttl_seconds else None,
            sha256=hashlib.sha256(data).hexdigest(),
        )
        self.index[artifact_id] = asdict(artifact)
        self._save_index()
        return artifact

    def get(self, artifact_id: str) -> Artifact:
        raw = self.index[artifact_id]
        if raw["expires_at"] and raw["expires_at"] < time.time():
            raise ValueError("artifact expired")
        return Artifact(**raw)

    def gc(self):
        now = time.time()
        removed = []
        for artifact_id, raw in list(self.index.items()):
            if raw["expires_at"] and raw["expires_at"] < now:
                Path(raw["path"]).unlink(missing_ok=True)
                removed.append(artifact_id)
                del self.index[artifact_id]
        self._save_index()
        return removed
```

然后工具不要返回大文件内容，而是返回 artifact 引用：

```python
def generate_report_tool(args):
    report_path = build_excel_report(args["date"])
    artifact = artifact_store.put_file(
        report_path,
        name=f"billing-{args['date']}.xlsx",
        mime_type="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    )
    return {
        "success": True,
        "artifact": {
            "id": artifact.id,
            "name": artifact.name,
            "mimeType": artifact.mime_type,
            "sizeBytes": artifact.size_bytes,
        },
        "llmHint": "不要把文件内容展开到消息里，直接把 artifact 交付给用户。",
    }
```

这就是最小可用版本。

---

## 3. pi-mono 生产版：ArtifactService + Storage Adapter

生产里不要把存储写死成本地磁盘。推荐抽象成 Service + Adapter。

```ts
interface ArtifactStorageAdapter {
  put(key: string, bytes: Buffer, contentType: string): Promise<void>;
  get(key: string): Promise<Buffer>;
  delete(key: string): Promise<void>;
  createSignedUrl?(key: string, expiresInSeconds: number): Promise<string>;
}

class ArtifactService {
  constructor(
    private storage: ArtifactStorageAdapter,
    private repo: ArtifactRepository,
  ) {}

  async create(input: {
    sessionId: string;
    name: string;
    mimeType: string;
    bytes: Buffer;
    ttlSeconds?: number;
  }): Promise<Artifact> {
    const id = `art_${crypto.randomUUID()}`;
    const sha256 = createHash("sha256").update(input.bytes).digest("hex");
    const key = `artifacts/${new Date().toISOString().slice(0, 10)}/${id}/${input.name}`;

    await this.storage.put(key, input.bytes, input.mimeType);

    return this.repo.insert({
      id,
      name: input.name,
      mimeType: input.mimeType,
      sizeBytes: input.bytes.length,
      storageKey: key,
      visibility: "private",
      ownerSessionId: input.sessionId,
      sha256,
      createdAt: new Date().toISOString(),
      expiresAt: input.ttlSeconds
        ? new Date(Date.now() + input.ttlSeconds * 1000).toISOString()
        : undefined,
    });
  }

  async getForSession(artifactId: string, sessionId: string): Promise<Artifact> {
    const artifact = await this.repo.findById(artifactId);
    if (!artifact) throw new Error("Artifact not found");
    if (artifact.ownerSessionId !== sessionId) throw new Error("Forbidden");
    if (artifact.expiresAt && Date.parse(artifact.expiresAt) < Date.now()) {
      throw new Error("Artifact expired");
    }
    return artifact;
  }
}
```

Adapter 可以接本地、S3、R2、GCS：

```ts
class S3ArtifactStorage implements ArtifactStorageAdapter {
  constructor(private s3: S3Client, private bucket: string) {}

  async put(key: string, bytes: Buffer, contentType: string) {
    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: bytes,
      ContentType: contentType,
    }));
  }

  async get(key: string) {
    const res = await this.s3.send(new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    }));
    return Buffer.from(await res.Body!.transformToByteArray());
  }

  async delete(key: string) {
    await this.s3.send(new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: key,
    }));
  }

  async createSignedUrl(key: string, expiresInSeconds = 300) {
    return getSignedUrl(
      this.s3,
      new GetObjectCommand({ Bucket: this.bucket, Key: key }),
      { expiresIn: expiresInSeconds },
    );
  }
}
```

---

## 4. OpenClaw 的落地方式：工具自动交付媒体

OpenClaw 里很多工具会把媒体结果自动 delivery，比如：

- `image_generate` 生成图片后自动返回 MEDIA 路径
- `tts` 生成音频后建议回复 `NO_REPLY`，避免重复消息
- `file_fetch` 可以从 node 拉取文件，保存在 gateway media store
- `message` 可以带 `media` / `buffer` / `filePath` 发送附件

这背后的思想也是 Artifact Store：

```text
工具生成文件 → 存入可控 media store → 返回引用 → 渲染层按渠道发送
```

Agent Loop 里可以把工具结果标准化：

```ts
type ToolResult =
  | { type: "text"; text: string }
  | { type: "artifact"; artifactId: string; name: string; mimeType: string }
  | { type: "auto_delivered"; channel: string; deliveryId: string };
```

如果工具已经自动发送了文件，就不要让 LLM 再发一遍：

```ts
if (result.type === "auto_delivered") {
  return {
    finalText: "NO_REPLY",
    reason: "Artifact was already delivered by the tool",
  };
}
```

这就是 OpenClaw `AUTO_DELIVER + NO_REPLY` 模式的工程含义。

---

## 5. Artifact 生命周期：生成、引用、交付、清理

一个完整生命周期建议这样设计：

```text
CREATE   工具生成文件，写入 Artifact Store
REFERENCE LLM 只拿 artifact id 和摘要，不拿完整 bytes
DELIVER  渲染层根据渠道能力发送附件或签名链接
RETENTION 根据 TTL / 合规要求保留
GC       定时清理过期文件和索引
AUDIT    记录谁生成、谁下载、谁发送
```

对应代码：

```ts
async function deliverArtifact(ctx: RunContext, artifactId: string) {
  const artifact = await artifactService.getForSession(artifactId, ctx.sessionId);

  if (ctx.channel.capabilities.attachments) {
    const bytes = await storage.get(artifact.storageKey);
    return ctx.channel.sendFile({
      filename: artifact.name,
      mimeType: artifact.mimeType,
      bytes,
    });
  }

  if (storage.createSignedUrl) {
    const url = await storage.createSignedUrl(artifact.storageKey, 5 * 60);
    return ctx.channel.sendText(`文件已生成：${artifact.name}\n下载链接（5分钟有效）：${url}`);
  }

  return ctx.channel.sendText(`文件已生成，但当前渠道不支持附件发送：${artifact.name}`);
}
```

---

## 6. 为什么不要把文件内容直接塞给 LLM？

因为文件经常是“大、脏、敏感、不可控”的：

```text
PDF 2MB  → 塞进上下文爆 token
Excel    → LLM 不适合读二进制
图片     → 要走 vision pipeline
日志包   → 先 grep / summarize，不要整包喂模型
私密报表 → 不应进入第三方模型上下文
```

更好的做法是：

```ts
return {
  artifact: {
    id: "art_123",
    name: "orders.xlsx",
    mimeType: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    sizeBytes: 381024,
  },
  summary: "订单报表已生成，包含 2026-05-01 的 1284 条订单记录。",
  availableActions: ["send_to_user", "summarize", "extract_rows"],
};
```

LLM 看到摘要和可用动作就够了。真要读文件，再调用专门工具。

---

## 7. 常见坑

### 坑 1：把本地绝对路径发给用户

```text
/Users/bot001/.openclaw/workspace/tmp/report.xlsx
```

用户打不开，其他机器也读不到。

✅ 返回 artifact id，由渠道层发送附件。

### 坑 2：公开 URL 永久有效

```text
https://cdn.example.com/private-report.xlsx
```

一旦泄露就长期可访问。

✅ 用短期 signed URL，默认 5-15 分钟。

### 坑 3：所有文件永久保存

Agent 跑几个月后磁盘爆炸。

✅ 默认 TTL：临时文件 24h，审计文件按合规策略保留。

### 坑 4：LLM 能访问所有 artifact

这会造成跨会话数据泄露。

✅ 每次读取检查 `ownerSessionId / userId / tenantId`。

### 坑 5：生成文件后没记录 hash

后续无法确认文件有没有被篡改。

✅ 存 `sha256`，审计/调试/合规都用得上。

---

## 8. 最小生产清单

做 Agent 文件系统，至少要有：

- [ ] `artifact_id`，不要直接暴露内部路径
- [ ] `mimeType`，方便不同渠道正确发送
- [ ] `sizeBytes`，防止超渠道限制
- [ ] `ownerSessionId/userId/tenantId` 权限校验
- [ ] `expiresAt` + GC 定时清理
- [ ] `sha256` 防篡改
- [ ] signed URL，短期有效
- [ ] delivery adapter：Telegram/Discord/Web/CLI 各自渲染
- [ ] 大文件只传摘要，按需读取
- [ ] 审计日志记录生成和发送行为

---

## 9. 一句话总结

**Artifact Store 是 Agent 的“文件后勤系统”：LLM 只负责理解和决策，文件的存储、权限、交付、清理由基础设施负责。**

没有 Artifact Store，Agent 只能做聊天机器人；有了 Artifact Store，Agent 才能稳定生成报表、图片、音频、补丁和交付物。