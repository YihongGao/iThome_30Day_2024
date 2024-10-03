# Day-29 實踐 DevSecOps - 自動化 Library 弱點掃描

# 前言
現代的軟體開發工作大多基於許多 Open-source 的 Library 之上，故當 Library 出現安全漏洞時，也會隨之受到影響。
攻擊者的手法也是日新月異，時常就會有新的弱點被揭露，即使是上個月或上週評估為安全的 Library 可能不再安全。

![https://snyk.io/_next/image/?url=https%3A%2F%2Fres.cloudinary.com%2Fsnyk%2Fimage%2Fupload%2Fv1663258115%2Fwordpress-sync%2Fblog-white-house-recs-iceberg.png&w=2560&q=75](https://snyk.io/_next/image/?url=https%3A%2F%2Fres.cloudinary.com%2Fsnyk%2Fimage%2Fupload%2Fv1663258115%2Fwordpress-sync%2Fblog-white-house-recs-iceberg.png&w=2560&q=75)
圖檔來源：[How Snyk helps satisfy White House cybersecurity recommendations](https://snyk.io/blog/snyk-white-house-cybersecurity-recommendations/)

今天將來介紹 [Dependency-Check] 這個專門掃描 Library 弱點的工具，並整合到 CI/CD Pipeline。

# Dependency-Check
**Dependency-Check** 是一套由 [OWASP（Open Web Application Security Project]）維護的專案。專門用來檢測軟體專案中的 Library 是否存在已知的安全漏洞，幫助開發者及安全團隊及時發現並修補潛在的安全風險。

能掃描多種語言的 Library 並產生弱點報告
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午8.09.02.51e4ktdsjs.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午8.09.02.51e4ktdsjs.webp)

## 工作原理
**Dependency-Check** 基於 [美國國家標準暨技術研究院 (National Institute of Standards and Technology)](https://www.nist.gov/) 發佈的 [國家漏洞資料庫(National Vulnerability Database)](https://nvd.nist.gov/) 為資料來源作為弱點的風險評分基準。

透過 CLI 與 Plugin 掃描軟體專案的 Library 資訊（如 build.gradle、pom.xml）與 CVE 清單進行比對後，產出弱點報告。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午8.42.58.4jo2w9k1x6.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午8.42.58.4jo2w9k1x6.webp)


## 使用範例

### 環境要求
依據 [官方文件](https://github.com/jeremylong/DependencyCheck?tab=readme-ov-file#requirements)
- **Minimum Java Version: Java 11**
- **(Highly Recommended & Option) NVD API Key**  
    能大幅加快下載 CVE 的速度，申請方式參考 [教學文件](https://github.com/jeremylong/DependencyCheck?tab=readme-ov-file#nvd-api-key-highly-recommended)

### 準備一個軟體專案
讀者能用自己的專案，或 直接 git clone 此 [Github 專案]

### 依據專案的套件管理工具進行配置 Plugin
若使用我提供的專案能跳過此步驟。

依 Maven 為例，再 pom.xml 添加以下設定即可，其他套件管理工具的 Plugin 安裝方式，可參考 [官方文件/Modules](https://jeremylong.github.io/DependencyCheck/modules.html) 
```xml
<project>
    ...
    <build>
        ...
        <plugins>
            ...
            <plugin>
              <groupId>org.owasp</groupId>
              <artifactId>dependency-check-maven</artifactId>
              <version>10.0.4</version>
              <executions>
                  <execution>
                      <goals>
                          <goal>check</goal>
                      </goals>
                  </execution>
              </executions>
            </plugin>
            ...
        </plugins>
        ...
    </build>
    ...
</project>
```

### 執行掃描
依 maven 為例，執行以下指令
```shell
# 有 NVD API Key 用此指令，會下載比較快
NVD_API_KEY=<你的 NVD API Key>
mvn verify -DnvdApiKey=$NVD_API_KEY

# 沒有 NVD API Key 則執行此指令
mvn verify 
```
>📘 第一次執行時，**Dependency-Check** 會將 CVE 資料下載到本地快取，所以可能會花費 5 ~ 10 分鐘，若有搭配 **NVD API Key** 能加快下載速度。

掃描完成應該能看到以下日誌
```shell
[INFO] Writing HTML report to: /Users/leokao/git/app-backend/target/dependency-check-report.html

# ...省略

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  7.614 s
[INFO] Finished at: 2024-10-03T21:21:17+08:00
[INFO] ------------------------------------------------------------------------
```

### 檢視掃描報告
開啟 `target/dependency-check-report.html` 的掃描報告
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午9.48.46.99tbuqpljj.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午9.48.46.99tbuqpljj.webp)
能看到 **Summary** 中列出了該軟體專案中哪些 Library 有弱點(綠框區塊)，並且標示出 風險等級(紅框區塊)。

點擊任何一個 Library 名稱，則會跳轉到該套件的弱點明細
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午9.53.16.86tmjuyq6r.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午9.53.16.86tmjuyq6r.webp)
這裡除了有該套件的基本資訊之外，還嘔 CVE 的說明與連結(紅框區塊) 與 第三方的參考資訊(綠框區塊)，節省人工另外查詢每個 CVE 的時間。

# 整合到 CI Pipeline
使用 [Github Action](https://github.com/features/actions) 建構出 Jar 之前進行弱點掃描，若有 CVSS 超過 8 的弱點時，中斷建構，並上傳弱點報告。

## 配置 Github Action workflow YML
>📘 讀者也能 fork 此 [Github 專案] 即可

新增 [build.yml](https://github.com/YihongGao/iThome_30Day_2024_dependency-check-demo/blob/main/.github/workflows/build.yml) 到專案目錄：`.github/workflows/`。

### build.yml 重點摘要
```yaml
  # .. 以上省略

  # 執行 dependency-check 的 job
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      # .. 省略
      - name: Analysis
        run: ./mvnw verify -DfailBuildOnCVSS=8
      # 若建構失敗，則上傳 dependency-check report
      - name: upload reports
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: reports
          path: |
            **/target/dependency-check-report.html

  build-jar:
    runs-on: ubuntu-latest
    # 需要 test job 與 dependency-check job 通過後才 build jar
    needs: [ test, dependency-check ]
  # ..以下省略
```
能看到 `dependency-check` 的 Job，使用 `./mvnw verify` 執行 **Dependency-Check** 的弱點掃描 並 加上 `-DfailBuildOnCVSS=8` 參數，代表掃出 CVSS 8分以上的弱點時，中斷 Pipeline。

### 掃描出 CVSS 8+ 時中斷 Pipeline
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午11.28.06.58hcgg3vuo.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241003/截圖-2024-10-03-下午11.28.06.58hcgg3vuo.webp)
開發者能從 **Artifacts** 區塊中下載弱點報告，進行弱點排除。

# 小結
今天介紹了如何使用 **Dependency-Check** 改善傳統低頻率且被動的弱點掃描計畫，將其自動化整合到 CI/CD Pipeline，以便在軟體專案建構過程中進行 Library 弱點掃描，避免弱點進入生產環境。這樣的方式不僅能強化了軟體開發流程中的安全防護，也有效減少了人工檢查的時間與成本。

# Refernce
- [Dependency-Check]
- [OWASP（Open Web Application Security Project]
- [How Snyk helps satisfy White House cybersecurity recommendations]
- [組み込んだOSSコンポーネントの更新漏れを可視化する「OWASP Dependency Check」](https://codezine.jp/article/detail/9608?p=2)


[Dependency-Check]: https://jeremylong.github.io/DependencyCheck/
[OWASP（Open Web Application Security Project]: https://owasp.org/

[How Snyk helps satisfy White House cybersecurity recommendations]:https://snyk.io/blog/snyk-white-house-cybersecurity-recommendations/

[Github 專案]: https://github.com/YihongGao/iThome_30Day_2024_dependency-check-demo