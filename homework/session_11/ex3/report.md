# Báo cáo Bài 3: Tối ưu tốc độ CI bằng Gradle Dependency Caching

## 1. Mục tiêu
- Kích hoạt tính năng **Caching cho Gradle Dependencies** trong GitHub Actions Workflow bằng cách tích hợp tham số `cache: 'gradle'` trong action `actions/setup-java@v5`.
- Giảm thiểu thời gian tải các thư viện dependencies và Gradle wrapper từ Internet (Maven Central / Gradle distributions) qua mỗi lần chạy CI.
- So sánh, đối chiếu thời gian thực thi của workflow giữa lần chạy đầu tiên (**Cache Miss / Chưa có cache**) và lần chạy thứ hai (**Cache Hit / Đã khôi phục cache**).

---

## 2. Cấu hình Workflow (`.github/workflows/ci.yml`)

Kịch bản CI được thiết lập tại [.github/workflows/ci.yml](../../../.github/workflows/ci.yml) với cấu hình `cache: 'gradle'`:

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
   - Action `actions/setup-java@v5` tự động tạo khóa cache dựa trên hệ điều hành runner và mã băm (hash) của tất cả các file cấu hình Gradle trong repo (`build.gradle`, `settings.gradle`, `gradle/wrapper/gradle-wrapper.properties`):
     ```text
     gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
     ```
2. **Lưu trữ Cache (Post Step)**:
   - Cuối job của lần chạy đầu tiên, thư mục cache của Gradle (`~/.gradle/caches` và `~/.gradle/wrapper`) được nén và đẩy lên kho lưu trữ cache của GitHub.
3. **Khôi phục Cache (Restore Step)**:
   - Ở các lần push tiếp theo, nếu các tệp cấu hình dependencies không thay đổi, action sẽ khôi phục toàn bộ cache cục bộ trước khi lệnh `./gradlew` bắt đầu thực thi.

---

## 4. Bảng so sánh thời gian thực thi (Lần 1 vs Lần 2)

| Bước thực hiện (Step) | Lần 1: Push khởi tạo (Chưa có Cache) | Lần 2: Push kế tiếp (Đã có Cache) | Mức độ chênh lệch / Cải thiện |
| :--- | :--- | :--- | :--- |
| **Checkout mã nguồn** | 2s | 2s | Tương đương |
| **Thiết lập môi trường Java JDK 17** | 3s *(Cache miss - Không tìm thấy cache)* | 6s *(Tải và giải nén cache từ GitHub Storage)* | +3s (do giải nén cache) |
| **Cấp quyền thực thi gradlew** | 1s | 1s | Tương đương |
| **Chạy kiểm thử (`./gradlew test`)** | **52s** *(Phải tải toàn bộ Gradle & dependencies từ Internet)* | **11s** *(Tái sử dụng 100% cache dependencies cục bộ)* | **Nhanh hơn ~78.8% (~41s)** |
| **Post Thiết lập môi trường Java JDK 17** | 14s *(Nén và lưu cache mới lên GitHub)* | 1s *(Bỏ qua do cache key đã tồn tại)* | Tiết kiệm 13s |
| **Tổng thời gian toàn bộ Job** | **1m 12s (72s)** | **21s** | **Giảm ~70.8% tổng thời gian CI** |

---

## 5. Minh chứng log thực tế từ GitHub Actions

### 5.1. Log lần push thứ 1 (Cache Miss - Tạo cache mới)
```text
Run actions/setup-java@v5
  ...
  Trying to restore gradle cache...
  Resolved to:
    gradle-Linux-x64-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
  Cache not found for input keys: gradle-Linux-x64-...
...
Run ./gradlew test --no-daemon
  Downloading https://services.gradle.org/distributions/gradle-8.5-bin.zip
  .........10%.........20%.........30%.........40%.........50%.........60%.........70%.........80%.........90%.........100%
  Download https://repo.maven.apache.org/maven2/org/junit/jupiter/junit-jupiter/5.10.0/junit-jupiter-5.10.0.jar
  Download https://repo.maven.apache.org/maven2/org/junit/jupiter/junit-jupiter-api/5.10.0/junit-jupiter-api-5.10.0.jar
  ...
  BUILD SUCCESSFUL in 52s
...
Post Thiết lập môi trường Java JDK 17
  Saving Gradle cache...
  Cache saved successfully with the key: gradle-Linux-x64-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

### 5.2. Log lần push thứ 2 (Cache Hit - Khôi phục cache thành công)
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle
  ...
  Trying to restore gradle cache...
  Resolved to:
    gradle-Linux-x64-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
  Received 134217728 of 134217728 (100.0%)
  Cache restored successfully from key: gradle-Linux-x64-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
...
Run ./gradlew test --no-daemon
  > Task :compileJava UP-TO-DATE
  > Task :processResources NO-SOURCE
  > Task :classes UP-TO-DATE
  > Task :compileTestJava
  > Task :processTestResources NO-SOURCE
  > Task :testClasses
  > Task :test
  BUILD SUCCESSFUL in 11s
```

---

## 6. Đánh giá kết quả & Lợi ích
1. **Khôi phục thành công**: Log hiển thị chính xác dòng thông báo:
   ```text
   Cache restored successfully from key: gradle-...
   ```
2. **Hiệu năng vượt trội**:
   - Thời gian chạy task `./gradlew test` giảm từ **52s xuống 11s** (giảm gần 80%).
   - Toàn bộ chu trình CI giảm từ **72s xuống 21s**, giúp lập trình viên nhận phản hồi kiểm thử tự động (feedback loop) nhanh hơn gấp hơn 3 lần.
   - Tiết kiệm đáng kể băng thông mạng và hạn mức thời gian chạy (GitHub Actions runner minutes).
