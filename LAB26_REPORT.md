# Lab26 — MCP và Function Calling

**Sinh viên:** Phạm Thanh Trung · **MSSV:** 2A202602949 · **Lớp:** K4

**Repo:** https://github.com/trump22/K4-DAY26-PhamThanhTrung-2A202602949

**Starter:** VinUni-AI20k/Day26-MCP-Tools-Integration, commit `d089197a8848a4fd3bcf64229bb9d313cdda3452`.

**Ngày thực nghiệm:** 14/10/2026, giờ Việt Nam. **Model:** `gemini-3.5-flash`, theo lựa chọn của sinh viên.

## Kết quả các bước bắt buộc

| Bước | Kết quả chạy thật | Bằng chứng |
|---|---|---|
| 01 — Function Calling | Gemini yêu cầu `get_weather` cho Hà Nội và Đà Nẵng; app thực thi hai hàm rồi Gemini tổng hợp câu trả lời tiếng Việt. Dữ liệu tool ở bước này là mock theo đề. | [Ảnh](screenshots/01-function-calling.png), [log](screenshots/logs/01-function-calling.log) |
| 02 — MCP stdio | `list_tools` khám phá `get_weather`; ba lần `call_tool` trả kết quả Hanoi, Danang, Haiphong. Không có LLM tham gia bước này. | [Ảnh](screenshots/02-mcp-client.png), [log](screenshots/logs/02-mcp-client.log) |
| 03a — Auth hợp lệ | Client khám phá và gọi tool qua Streamable HTTP, trả `Hanoi: 29°C, trời mưa`. | [Ảnh](screenshots/03a-auth-ok.png), [log](screenshots/logs/03a-auth-ok.log) |
| 03a — Auth sai | `wrong-token` bị từ chối trước khi khám phá tool. Server thực tế ghi `POST /mcp HTTP/1.1 401 Unauthorized`; client exit 1 là kết quả âm tính mong đợi. | [Ảnh](screenshots/03a-auth-fail.png), [log](screenshots/logs/03a-auth-fail.log) |
| 03b — Registry | Tìm được hai tool có tag weather; chọn `get_weather_v2 v2.0.0` và gọi server được registry chỉ định. | [Ảnh](screenshots/03b-registry.png), [log](screenshots/logs/03b-registry.log) |
| 03c — Versioning | Resource metadata thông báo v1 deprecated; v1 vẫn trả chuỗi, v2 trả JSON với `api_version=2.0` và forecast. | [Ảnh](screenshots/03c-versioning.png), [log](screenshots/logs/03c-versioning.log) |
| 04 — Setup | Server WeatherAPI chạy Streamable HTTP ở `:8085/mcp`; script kiểm tra trả `All checks passed`, model 3.5 Flash. | [Ảnh](screenshots/04-verify-setup.png), [log](screenshots/logs/04-verify-setup.log) |
| 04 — ADK current | Agent gọi `get_current_weather("Hanoi")`, nhận dữ liệu thật và tổng hợp câu trả lời. | [Ảnh chat](screenshots/04-adk-current.png), [events thật](screenshots/logs/04-adk-session.json) |
| 04 — ADK forecast | Agent gọi `get_forecast("Tokyo", days=3)`, trả đủ ba ngày với nhiệt độ cao/thấp và xác suất mưa. | [Ảnh chat](screenshots/04-adk-forecast.png), [events thật](screenshots/logs/04-adk-session.json) |

Hai ảnh ADK chụp trực tiếp từ giao diện. Các ảnh output dòng lệnh hiển thị log stdout/stderr gốc trên trình duyệt; đây là ảnh của log đã ghi, không phải cửa sổ terminal tương tác. Log đầy đủ được nộp kèm để đối chiếu. Không tạo dữ liệu chạy giả. File session xuất từ API của ADK sau hai lượt chat; bỏ metadata chữ ký suy luận, giữ nguyên text và function call/response.

## Dữ liệu WeatherAPI thật đã nhận

Hanoi, cập nhật **2026-10-14 19:00**: nhiệt độ **29.4°C**, cảm giác như **32.2°C**, Overcast, độ ẩm **63%**, UV **0.0**, gió **18.4 km/h SSE**, áp suất **1013 mb**, tầm nhìn **7.5 km**.

| Tokyo | Cao nhất | Thấp nhất | Khả năng mưa |
|---|---:|---:|---:|
| 14/10/2026 | 28.1°C | 20.7°C | 36% |
| 15/10/2026 | 21.8°C | 18.6°C | 7% |
| 16/10/2026 | 24.9°C | 19.8°C | 14% |

Đối chiếu function response với câu trả lời cuối trong session export cho thấy agent dùng kết quả MCP, không chạy agent fallback không có tool. `verify_setup.py` nhận HTTP 406 khi GET thuần; đúng hướng dẫn của Lab, đó là endpoint sống nhưng yêu cầu MCP headers/session. Hai tool call thật mới xác nhận đầy đủ kết nối MCP.

## Các lỗi, nguyên nhân và điều chỉnh

1. **Model 2.5 Flash: HTTP 404.** Provider báo model của starter không còn khả dụng cho tài khoản. Đổi model mặc định sang **3.5 Flash theo yêu cầu sinh viên** và cho phép đặt `GEMINI_MODEL`; không đổi schema, mock data hay vòng lặp tool. Lần thử 3.8 Flash trước khi nhận lựa chọn của sinh viên bị HTTP 503, không dùng cho bằng chứng thành công.
2. **HTTP 503 trên 3.5 Flash.** Request đầu gọi được hai tool nhưng request tổng hợp bị provider báo high demand. Chạy lại cùng model, cùng script và cùng câu hỏi thành công. Không nhầm lỗi quá tải 503 với lỗi quota 429; không thêm retry vô hạn.
3. **Registry: UnicodeDecodeError trên Windows.** Python dùng encoding mặc định cp1252 để đọc JSON tiếng Việt. Đặt `PYTHONUTF8=1` và `PYTHONIOENCODING=utf-8`; giữ nguyên registry/code.
4. **Auth sai: 401 thay vì 403 được mô tả trong hướng dẫn.** SDK MCP 2.3.0 từ chối bearer không hợp lệ bằng 401. Đây là hành vi thực tế của transport, không phải tool chạy thành công hay lỗi cần bỏ auth. Giữ nguyên xác thực.
5. **`verify_setup.py` in 10 ký tự đầu key.** Chỉ đổi dòng in thành `configured (value hidden)` để ảnh không lộ key; giữ nguyên mọi điều kiện kiểm tra. Không sửa hoặc thêm test để đổi kết quả.
6. **Cảnh báo thư viện.** `google.generativeai` trong script kiểm tra dependency đã deprecated; chương trình gọi model thực tế dùng `google.genai`/ADK. Auth demo có cảnh báo cấu hình audience của MCP SDK. Đây là demo token tĩnh local, không tuyên bố sẵn sàng triển khai bảo mật production.
7. **Ảnh chụp trình duyệt.** Công cụ có lúc timeout và trả vùng trống rất lớn. Chỉ cắt vùng trống quanh ảnh bằng chứng và chuyển ảnh sang PNG; không sửa nội dung câu hỏi/câu trả lời.

Auth server ở cổng 8000 đã dừng sau demo để giải phóng cổng cho ADK. MCP WeatherAPI chạy ở 8085; ADK web ở 8000. Dùng biến `PORT=8085` để server chọn Streamable HTTP khi chạy từ shell không có TTY, đúng nhánh có sẵn trong starter.

## Chạy lại trên Windows

Python **3.13.15** (đáp ứng >=3.12). Bước 01–03 dùng `.venv` ở gốc: MCP **2.3.0**, google-genai **2.30.1**. Lab04 dùng hai môi trường `uv sync` riêng từ lockfile starter: MCP **1.28.1**, FastMCP **3.4.2**, ADK **2.3.0**. Không trộn hai môi trường.

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt uv
$env:PYTHONUTF8 = '1'
$env:PYTHONIOENCODING = 'utf-8'
$env:GEMINI_MODEL = 'gemini-3.5-flash'
# Đặt GEMINI_API_KEY bằng key riêng, không in key hay commit .env.
.\.venv\Scripts\python.exe 01-function-calling\weather_function_calling.py
# Các client stdio chạy từ thư mục chứa script, dùng Python của .venv gốc.
```

Hai terminal cho Lab04:

```powershell
# Terminal server: vào 04-lab/mcp-server
..\..\.venv\Scripts\uv.exe sync
$env:PORT = '8085'
# Đặt WEATHERAPI_KEY trong terminal server.
..\..\.venv\Scripts\uv.exe run python weather.py

# Terminal client: vào 04-lab/mcp-client
# Sao chép .env.example thành .env, nhập GOOGLE_API_KEY riêng; lưu UTF-8.
..\..\.venv\Scripts\uv.exe sync
..\..\.venv\Scripts\uv.exe run python verify_setup.py
..\..\.venv\Scripts\uv.exe run adk web --host 127.0.0.1 --port 8000
```

## Phạm vi và bảo vệ key

Hoàn thành các bước bắt buộc chạy local. **Cloud Run là phần tùy chọn, không triển khai**, không tạo service hay phát sinh chi phí cloud. Không cần Docker cho các bước bắt buộc.

`.env` ở gốc chứa key local; `.env` client chứa Gemini key. Hai file được `.gitignore` bỏ qua. Chỉ commit `.env.example` chứa placeholder; không ghi key vào mã, README, ảnh hay log. Các thư mục 01–04, registry, Dockerfile và cấu trúc starter được giữ. Chỉ chỉnh model và dòng log che key, bổ sung bằng chứng/tài liệu.

Codex hỗ trợ thực hiện lệnh, chọn cấu hình theo yêu cầu sinh viên, thu thập bằng chứng và viết báo cáo. Kết quả trong tài liệu lấy từ các tiến trình và phiên chat thật; sinh viên cần đọc lại và hiểu nội dung trước khi được đánh giá.
