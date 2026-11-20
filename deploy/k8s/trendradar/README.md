# TrendRadar dashboard trên Kubernetes

Triển khai [TrendRadar](https://github.com/sansan0/TrendRadar) (GPL-3.0) lên Kubernetes bằng Kustomize,
dùng image chính thức `wantcat/trendradar:6.11.0` và `wantcat/trendradar-mcp:4.1.1`.

## Kiến trúc

```
Ingress (trendradar.example.com) ──► Service :80 ──► Pod trendradar
                                     Service :3333 ─┐   ├─ container trendradar: crawl theo CRON_SCHEDULE
                                     (nội bộ)       │   │   + dashboard HTML tĩnh :8080 (/app/output)
                                                    └──►└─ container mcp: MCP server HTTP :3333
                                                        PVC trendradar-output ─► /app/output (SQLite, HTML)
                                                        ConfigMap ─► /app/config (read-only)
```

- 1 replica, strategy `Recreate`: SQLite chỉ cho phép một tiến trình ghi, PVC là `ReadWriteOnce`.
- MCP chạy chung pod để dùng chung volume, không cần storage `ReadWriteMany`.
- Ingress chỉ mở dashboard; cổng MCP chỉ truy cập được trong cluster.

## Triển khai

```bash
cd deploy/k8s/trendradar

# 1. Tạo file secret (bắt buộc, có thể để trống hết)
cp secret.env.example secret.env
$EDITOR secret.env

# 2. Sửa host trong ingress.yaml (và ingressClassName nếu không dùng nginx)

# 3. Xem trước rồi apply
kubectl kustomize . | less
kubectl apply -k .

# 4. Theo dõi
kubectl -n trendradar rollout status deploy/trendradar
kubectl -n trendradar logs deploy/trendradar -c trendradar -f
```

Lần khởi động đầu tiên crawl ngay một lượt (`IMMEDIATE_RUN=true`) trước khi bật web server,
nên pod có thể mất 1–3 phút mới Ready.

Chưa có Ingress controller? Truy cập tạm bằng port-forward:

```bash
kubectl -n trendradar port-forward svc/trendradar 8080:80     # dashboard: http://localhost:8080
kubectl -n trendradar port-forward svc/trendradar 3333:3333   # MCP: http://localhost:3333/mcp
```

## Cấu hình

| Muốn đổi | Sửa ở đâu |
|---|---|
| Nguồn tin, từ khoá, lịch đẩy tin, AI | `config/config.yaml`, `config/frequency_words.txt`, `config/timeline.yaml` |
| Chu kỳ crawl, múi giờ container | `configMapGenerator` → `trendradar-env` trong `kustomization.yaml` |
| Webhook, AI key, S3 | `secret.env` |
| Phiên bản image | `images:` trong `kustomization.yaml` |
| Dung lượng lưu trữ / StorageClass | `pvc.yaml` |

Sau khi sửa, chạy lại `kubectl apply -k .`. Kustomize thêm hash vào tên ConfigMap/Secret
nên pod sẽ tự khởi động lại với cấu hình mới.

### Cấu hình hiện tại: tin hot bằng tiếng Việt

Các file trong `config/` lấy từ upstream v6.11.0, đã chỉnh như sau:

- **Chủ đề:** AI, thương mại điện tử, ẩm thực, học tập/giáo dục, đời sống, công nghệ tiêu dùng.
  - `filter.method: "ai"`: AI đọc `config/ai_interests.txt` (viết bằng tiếng Việt) để lọc tin.
    Muốn thêm/bớt chủ đề thì sửa file đó.
  - `config/frequency_words.txt`: bộ từ khoá dự phòng (Trung/Việt/Anh) cho cùng 5 nhóm,
    dùng khi đổi `filter.method` về `"keyword"` (không tốn token) và cho công cụ MCP `get_trending_topics`.
- **Dịch:** `ai_translation.language: "Tiếng Việt"`, dịch cả tiêu đề hot list (`scope.hotlist: true`) lẫn RSS.
- **Phân tích AI:** `ai_analysis.language: "Vietnamese"`, có tính cả tin RSS.
- **Nguồn tin:** giữ các hot list Trung Quốc (Weibo, Douyin, Zhihu, Baidu…), thêm RSS VnExpress
  (Số hóa, Kinh doanh, Giáo dục, Đời sống, Du lịch, Sức khỏe), Tuổi Trẻ (Nhịp sống số, Giáo dục),
  TechCrunch (AI, E-commerce), The Verge AI, Hacker News.
- **Múi giờ:** `app.timezone: Asia/Ho_Chi_Minh`.

**Bắt buộc có AI key** cho lọc AI, dịch và phân tích: điền `AI_API_KEY` (và `AI_MODEL`,
`AI_API_BASE` nếu không dùng DeepSeek mặc định) trong `secret.env`. Không có key thì đặt
`filter.method: "keyword"` và tắt `ai_translation.enabled` / `ai_analysis.enabled`.

Giao diện dashboard (chữ cố định trong HTML của TrendRadar) vẫn là tiếng Trung;
chỉ nội dung tin và phân tích là tiếng Việt.

## Basic auth

Dashboard không có đăng nhập, và web server phục vụ toàn bộ `/app/output` (bao gồm file
SQLite). Nếu mở ra Internet, hãy bật basic auth (ingress-nginx):

```bash
htpasswd -c auth admin
kubectl -n trendradar create secret generic trendradar-basic-auth --from-file=auth
```

rồi bỏ comment các annotation `nginx.ingress.kubernetes.io/auth-*` trong `ingress.yaml`.

## Gỡ bỏ

```bash
kubectl delete -k .          # xoá cả namespace và PVC (mất dữ liệu)
```
