# Báo cáo Bài 3: Tối ưu tốc độ CI bằng Gradle Dependency Caching

## 1. Mục tiêu
- Kích hoạt tính năng **Caching cho Gradle Dependencies** trong GitHub Actions Workflow bằng cách tích hợp tham số `cache: 'gradle'` trong action `actions/setup-java@v5`.
- Giảm thiểu thời gian tải các thư viện dependencies và Gradle wrapper từ Internet qua mỗi lần chạy CI.
- Kiểm chứng và so sánh thời gian thực thi của workflow giữa lần chạy đầu tiên (**Cache Miss / Chưa có cache**) và lần chạy thứ hai (**Cache Hit / Đã khôi phục cache thành công**).

---

## 2. Kịch bản CI (`.github/workflows/ci.yml`)

Kịch bản workflow hoàn chỉnh được thiết lập tại [.github/workflows/ci.yml](../../../.github/workflows/ci.yml):

```yaml
name: Java CI with Gradle Caching

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v4

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền thực thi cho Gradle wrapper
        run: chmod +x gradlew

      - name: Chạy kiểm thử tự động với Gradle
        run: ./gradlew test --no-daemon
```

---

## 3. Cơ chế hoạt động của Gradle Dependency Cache

1. **Khóa Cache (Cache Key)**:
   - Action `actions/setup-java@v5` tự động tạo khóa cache dựa trên hệ điều hành runner và hash của các file cấu hình Gradle (`build.gradle`, `settings.gradle`, `gradle-wrapper.properties`):
     ```text
     gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
     ```
2. **Lưu trữ Cache (Post Step)**:
   - Cuối job của lần chạy đầu tiên, thư mục `~/.gradle/caches` và `~/.gradle/wrapper` được nén và đẩy lên GitHub Actions Cache storage (`Post Thiết lập môi trường Java JDK 17`).
3. **Khôi phục Cache (Restore Step)**:
   - Ở lần push thứ 2, action phát hiện cache key trùng khớp và tự động tải, giải nén cache về máy chủ runner trước khi lệnh `./gradlew` bắt đầu thực thi.

---

## 4. Bảng so sánh thời gian thực thi thực tế (Run 1 vs Run 2)

> **Thông tin thực thi trên GitHub Actions:**
> - **Lần 1 (Run ID: 38252367361)**: Commit khởi tạo `26cf6c8` (Chưa có cache)
> - **Lần 2 (Run ID: 38252675430)**: Commit kế tiếp `9e385cc` (Đã có cache)

| Bước thực hiện (Step) | Lần 1: Chưa có Cache (Run 1) | Lần 2: Đã có Cache (Run 2) | Mức độ tối ưu / Đánh giá |
| :--- | :---: | :---: | :--- |
| **Set up job** | 1s | 2s | Tương đương |
| **Checkout mã nguồn** | 0s | 0s | Tương đương |
| **Thiết lập môi trường Java JDK 17** | 1s | 4s | +3s (do thực hiện tải và giải nén cache từ GitHub Storage) |
| **Cấp quyền thực thi cho Gradle wrapper** | 0s | 0s | Tương đương |
| **Chạy kiểm thử tự động với Gradle (`./gradlew test`)** | **15s** | **5s** | **Nhanh gấp 3 lần (Giảm 66.7% thời gian kiểm thử)** |
| **Post Thiết lập môi trường Java JDK 17** | 3s | 0s | Tiết kiệm 3s (bỏ qua bước lưu do cache key đã tồn tại) |
| **Post Checkout mã nguồn / Complete job** | 0s | 1s | Tương đương |
| **Tổng thời gian chạy các bước** | **20s** | **12s** | **Giảm 40% tổng thời gian thực thi** |

---

## 5. Minh chứng log thực tế từ GitHub Actions

### 5.1. Log Lần 1 (Run ID 38252367361 - Cache Miss)
- Bước `Thiết lập môi trường Java JDK 17`:
  ```text
  Trying to restore gradle cache...
  Resolved to:
    gradle-Linux-x64-c2c2f6277fbc286c0e86b2faee44d82ef2b1b369ba90b9b3df9bbbaad6ba8296
  Cache not found for input keys: gradle-Linux-x64-...
  ```
- Bước `Chạy kiểm thử tự động với Gradle`:
  ```text
  Downloading https://services.gradle.org/distributions/gradle-8.5-bin.zip
  ... (Tải Gradle wrapper và dependencies JUnit 5 từ Maven Central) ...
  BUILD SUCCESSFUL in 15s
  ```
- Bước `Post Thiết lập môi trường Java JDK 17`:
  ```text
  Saving Gradle cache...
  Cache saved successfully with the key: gradle-Linux-x64-c2c2f6277fbc286c0e86b2faee44d82ef2b1b369ba90b9b3df9bbbaad6ba8296
  ```

### 5.2. Log Lần 2 (Run ID 38252675430 - Cache Hit)
- Bước `Thiết lập môi trường Java JDK 17`:
  ```text
  Trying to restore gradle cache...
  Resolved to:
    gradle-Linux-x64-c2c2f6277fbc286c0e86b2faee44d82ef2b1b369ba90b9b3df9bbbaad6ba8296
  Received 63939525 of 63939525 (100.0%)
  Cache restored successfully from key: gradle-Linux-x64-c2c2f6277fbc286c0e86b2faee44d82ef2b1b369ba90b9b3df9bbbaad6ba8296
  ```
- Bước `Chạy kiểm thử tự động với Gradle`:
  ```text
  > Task :compileJava UP-TO-DATE
  > Task :processResources NO-SOURCE
  > Task :classes UP-TO-DATE
  > Task :compileTestJava
  > Task :processTestResources NO-SOURCE
  > Task :testClasses
  > Task :test
  BUILD SUCCESSFUL in 5s
  ```

---

## 6. Kết luận
- Việc tích hợp `cache: 'gradle'` trong `actions/setup-java@v5` đã giải quyết triệt để vấn đề tải lặp đi lặp lại các thư viện dependencies qua mạng mỗi lần CI chạy.
- Kết quả kiểm chứng log ghi nhận chính xác: `Cache restored successfully from key: gradle-...`.
- Tốc độ chạy bước kiểm thử `./gradlew test` cải thiện rõ rệt từ **15s xuống còn 5s**, tối ưu đáng kể tài nguyên runner và chu kỳ phản hồi mã nguồn.
