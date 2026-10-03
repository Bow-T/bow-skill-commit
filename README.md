# bow-skill-commit

Bộ skill **ghép** (add-on) cho [bow-agent](https://github.com/Bow-T/bow-agent): quy ước
**commit / nhánh / push** của team Bow, dùng được với **mọi stack**. Tách ra từ
`bow-skill-flutter` để stack nào cũng có — trước đây chỉ Flutter + Supabase mang `bow-commit`,
các stack khác (Flutter + API, Next.js, React Native) không có quy ước commit nào.

## Cấu trúc

```
bow-skill-commit/
├── bow-skill.json          # manifest — id "commit", skillsDir "skills"
└── skills/
    └── bow-commit/         # commit + push chuẩn Bow, không phụ thuộc công nghệ
```

## `bow-commit` làm gì

Ba luật cứng, repo nào cũng áp:

1. **Không trailer ghi công AI** (`Co-Authored-By: Claude…`, `Generated with…`) — cả commit lẫn mô tả MR/PR.
2. **Không commit thẳng lên nhánh chính** (`main`/`master`/`develop`/`dev`) — tách nhánh
   `<type>/<MÃ-JIRA>-<số>-<slug>` trước.
3. **Không vượt cổng** — không `--no-verify`, không force-push, không stage secret / file cấu hình máy.

Phần còn lại tự thích nghi theo repo:

| Repo có gì | Skill làm gì |
| --- | --- |
| script cổng `scripts/*-commit.sh` đã commit ở gốc repo (tên do repo đặt) | mở ra xem nó chỉ KIỂM (không tự `git add` / `commit` / `push`) rồi chạy từ gốc repo làm cổng chặn (safety scan + analyzer + checklist) |
| không có script | tự soát diff đã stage (secret, CORS wildcard, đồ debug sót) rồi chạy check của chính repo: `flutter analyze` / `typecheck` · `lint` · `test` của `package.json` |
| `scripts/quest-checklist.sh` | bắt buộc tự chấm rubric trước khi push |
| quy ước commit riêng (CLAUDE.md / CONTRIBUTING) | theo quy ước của repo; ba luật cứng vẫn giữ |

Khuôn message mặc định: `Type(scope): <gitmoji> Subject` · body ≥ 3 dòng giải thích VÌ SAO ·
footer `Refs: <MÃ-JIRA>-<số>`.

**Dự án được nhận bằng MÃ JIRA** (`ABC` trong `ABC-123`), không bằng tên sản phẩm hay tên repo: skill
tự dò mã từ tên nhánh, không có thì lấy tiền tố xuất hiện nhiều nhất trong 20 commit gần nhất, cuối
cùng mới hỏi. Skill không gắn cứng tên dự án nào.

## Tự nhận diện

`targets.detect` khai `{ "path": ".git" }`: repo git nào cũng khớp, nên bow-agent tự tick bộ này cho
mọi dự án chưa khai công nghệ — quy ước commit không phụ thuộc repo viết bằng gì. Không muốn thì bỏ
tick ở ô Công nghệ (mục "Dùng chung mọi công nghệ").

## Đăng ký vào bow-agent

Bộ ghép **không** nằm trong `stacks` (không hiện ở dropdown chọn công nghệ). Khai ở `addons`
của `~/.bow-agent/registry.json`, rồi thêm `with` vào stack nào cần:

```json
{
  "addons": [
    { "id": "commit", "label": "Commit chuẩn Bow", "repo": "github.com/Bow-T/bow-skill-commit", "ref": "v1.2.0" }
  ],
  "stacks": [
    { "id": "flutter-api", "label": "Flutter + API", "repo": "github.com/Bow-T/bow-skill-flutter-api", "ref": "v1.0.0", "with": ["commit"] }
  ]
}
```

Chọn stack có `with: ["commit"]` là bow trải luôn `bow-commit` vào `.claude/skills/` của dự án.
Trùng tên skill thì **stack thắng**: stack còn tự mang `bow-commit` (bản riêng) thì bản đó được
dùng, bộ ghép nhường. Repo đích đã có bản fork riêng thì khai tên vào `.claude/skills/.bow-skip`.

Thử local trước khi đẩy GitHub: `"repo": "file:///đường/dẫn/tới/bow-skill-commit"`.

## Phát hành phiên bản mới

Sửa skill → tăng `version` trong `bow-skill.json` → gắn tag (`git tag v1.0.1 && git push --tags`)
→ cập nhật `ref` của mục `commit` trong registry. bow luôn tải theo tag và cache theo `<id>@<ref>`,
nên sửa nội dung mà giữ nguyên tag sẽ KHÔNG được tải lại.
