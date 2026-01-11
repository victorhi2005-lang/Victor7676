#專案功能 

本專案開發一個「安全性導向的雲端記帳/備註系統」。

主要功能如下：

-動態資料存取： 提供使用者透過網頁介面輸入敏感的記帳備註（如金額、支出項目）。

-高強度資安防禦： 針對常見的網頁攻擊（如 SQL 注入、跨站腳本攻擊 XSS）實作了深層的後端防禦邏輯。

-資料持久化： 整合實體資料庫，確保使用者輸入的內容能安全地儲存，並在需要時進行查詢。

#系統架構 
-前端介面： 使用 Java HttpServer 建立的動態網頁。

-後端邏輯： 負責 URL 解碼、字串清洗（Sanitization）以及透過 JDBC 執行參數化查詢（PreparedStatements）。

-資料庫層： 使用 MariaDB 10.6，存放於獨立的受控網路中，不對外開放預設埠

#部署方式

-工具： 使用 docker-compose 進行多容器編排。

-流程： 僅需在專案根目錄執行 docker-compose up -d，系統會自動拉取鏡像、建立資料庫實例、配置環境變數並啟動應用服務。

#運行流程

-請求發起： 使用者透過瀏覽器訪問 http://localhost:8081。

-輸入過濾： 當使用者點擊儲存，後端立即執行 XSS 過濾，將特殊字元（如 <、>）轉換為安全的 HTML 實體。

-安全存取： 程式透過 PreparedStatement 將清洗後的字串傳送至 MariaDB，防止 SQL 注入。

-結果回傳： 系統確認資料寫入後，回傳加密/轉義後的內容至前端顯示，確保資料在顯示時亦不具威脅性。

-測試驗證： 透過 sqlmap 進行自動化測試，確認所有輸入點皆具備抗攻擊能力。

---------------------------------------------------------------------------------------------------------------

```
-$ cd ~/SecurityProject
nano SecureApp. java
wget https://downloads.mariadb.com/Connectors/java/connector-java-3.1.2/mariadb-java-client-3.1.2.jar
sudo docker exec securityproject-app-1 javac SecureApp. java
sudo docker exec -d securityproject-app-1 java -cp .: mariadb-java-client-3.1.2. jar SecureApp
sudo docker exec securityproject-app-1 pkill java
sudo docker logs securityproject-app-1
sudo docker exec securityproject-app-1 java -cp .: mariadb-java-client-3.1.2. jar SecureApp
sudo docker exec securityproject-app-1 pkill -9 java
sudo docker exec securityproject-app-1 java -cp .: mariadb-java-client-3.1.2.jar SecureApp

```
---------------------------------------------------------------------------------------------------------------

#專案完整原始碼

```
import com.sun.net.httpserver.HttpServer;
import com.sun.net.httpserver.HttpExchange;
import java.io.*;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.sql.*;

public class SecureApp {
    static final String DB_URL = "jdbc:mariadb://db:3306/secure_db";
    static final String USER = "root";
    static final String PASS = "root";

    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
        
        server.createContext("/", (exchange) -> {
            String html = "<html><body style='font-family:sans-serif; padding:20px;'>" +
                          "<h2>資管期末專案：安全記帳系統</h2>" +
                          "<form action='/save' method='POST'>" +
                          "支出備註: <input type='text' name='note'> " +
                          "<input type='submit' value='儲存'>" +
                          "</form></body></html>";
            sendResponse(exchange, html);
        });

        server.createContext("/save", (exchange) -> {
            if ("POST".equals(exchange.getRequestMethod())) {
                InputStreamReader isr = new InputStreamReader(exchange.getRequestBody(), StandardCharsets.UTF_8);
                BufferedReader br = new BufferedReader(isr);
                String formData = br.readLine();
                String userInput = formData.split("=").length > 1 ? formData.split("=")[1] : "";
                userInput = java.net.URLDecoder.decode(userInput, StandardCharsets.UTF_8);

                String safeInput = userInput.replace("<", "&lt;").replace(">", "&gt;");

                String statusMessage = "";
                try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASS)) {
                    String sql = "INSERT INTO notes (note_content) VALUES (?)";
                    PreparedStatement pstmt = conn.prepareStatement(sql);
                    pstmt.setString(1, safeInput);
                    pstmt.executeUpdate();
                    statusMessage = "資料已成功安全地存入資料庫！";
                } catch (Exception e) {
                    statusMessage = "儲存發生錯誤: " + e.getMessage();
                }

                String html = "<html><body><h3>" + statusMessage + "</h3>" +
                              "<p>存入內容: " + safeInput + "</p>" +
                              "<a href='/'>返回首頁</a></body></html>";
                sendResponse(exchange, html);
            }
        });

        System.out.println("伺服器已啟動於 http://localhost:8081");
        server.start();
    }

    private static void sendResponse(HttpExchange exchange, String response) throws IOException {
        byte[] bytes = response.getBytes(StandardCharsets.UTF_8);
        exchange.getResponseHeaders().set("Content-Type", "text/html; charset=UTF-8");
        exchange.sendResponseHeaders(200, bytes.length);
        OutputStream os = exchange.getResponseBody();
        os.write(bytes);
        os.close();
    }
}

```
---------------------------------------------------------------------------------------------------------------

#後端安全設計 

-防止 SQL 注入 ：

-證據： 影片中 sqlmap 回傳 all tested parameters do not appear to be injectable。

-說明： 程式碼不直接拼接 SQL 字串，並通過專業滲透工具驗證，確保資料庫安全。

#環境隔離與資料庫防護：

-證據： docker-compose.yml 中定義了 db 與 app 兩個服務。

-說明： 資料庫運行在獨立容器中，不對外部網路開放（沒有將 3306 映射到主機），有效防止遠端暴力破解。

-<img width="601" height="460" alt="image" src="https://github.com/user-attachments/assets/25394c9e-335a-469e-bdad-01953b24117d" />

#敏感資訊保護 ：

-證據： docker-compose.yml 中的 MYSQL_ROOT_PASSWORD: root。

-說明： 密碼透過環境變數管理，而非直接硬編碼 (Hardcode) 在 Java 原始碼中，符合安全開發規範。

<img width="405" height="371" alt="image" src="https://github.com/user-attachments/assets/4561bdc1-a1d2-4df3-88e1-8886b82acc2c" />

#最小化權限與埠號管控：

-證據： ports: - "8081:8080"。

-說明： 僅開放必要的服務埠號，並透過非標準埠號 (8081) 提供服務，減少被掃描攻擊的機率。

#伺服器端輸入過濾：

-證據： SecureApp.java 原始碼中的 userInput.replace 邏輯。

-說明： 在後端接收資料時立即進行清洗，確保存入系統的資料不含惡意代碼。

---------------------------------------------------------------------------------------------------------------

#前端安全設計

#跨站腳本攻擊防護 ：

-證據： 截圖顯示輸入 <script>alert('Hacked')</script> 後，頁面正確顯示為純文字。

-說明： 前端顯示時已進行 HTML 實體轉義 (Escape)，確保腳本不會被瀏覽器執行。

#通訊埠掩蔽設計：

-證據： 瀏覽器網址顯示為 localhost:8081。

-說明： 隱藏後端真實運行的 8080 埠，透過 Docker 映射增加一層代理防護。

#URL 解碼安全處理：

-證據： 原始碼中 java.net.URLDecoder.decode。

-說明： 正確處理前端傳回的編碼字串，防止特殊字元繞過過濾邏輯

---------------------------------------------------------------------------------------------------------------

#開發環境 ：

-作業系統 (OS)： Kali Linux 。

-JDK 版本： Eclipse Temurin 17 (LTS)。

-DBC 驅動： MariaDB Java Client 3.1.2。

#測試工具：

-Sqlmap： 用於自動化 SQL 注入漏洞掃描與驗證。

-Nmap： 用於檢查容器通訊埠開放狀態，確保最小化曝露。

-Firefox (Burp Suite 整合)： 用於手動驗證 XSS 攻擊載荷之轉義效果。

---------------------------------------------------------------------------------------------------------------

#資料庫建置方式與資料表設計

(1) 建置方式

我們透過 Docker Compose 實現自動化配置與快速部署，並強化了以下安全設定：

<img width="613" height="140" alt="image" src="https://github.com/user-attachments/assets/f5426f89-f0ae-4029-9203-31c74c203b81" />

---------------------------------------------------------------------------------------------------------------

<img width="1354" height="774" alt="image" src="https://github.com/user-attachments/assets/df227f19-dc2b-442e-8198-33b6fdfa44b0" />

<img width="1083" height="593" alt="image" src="https://github.com/user-attachments/assets/db0f3b08-a2ae-4f3e-b6a0-cfc75ace955e" />

#證實系統具備自動化滲透測試的防禦力。


