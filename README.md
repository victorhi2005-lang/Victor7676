1. 後端安全設計 
• 防止 SQL 注入 (SQLi Prevention)：
• 證據： 影片中 sqlmap 回傳 all tested parameters do not appear to be injectable。
• 說明： 程式碼不直接拼接 SQL 字串，並通過專業滲透工具驗證，確保資料庫安全。
• 環境隔離與資料庫防護：
• 證據： docker-compose.yml 中定義了 db 與 app 兩個服務。
• 說明： 資料庫運行在獨立容器中，不對外部網路開放（沒有將 3306 映射到主機），有效防止遠端暴力破解。
• 敏感資訊保護 (No Hardcoding)：
• 證據： docker-compose.yml 中的 MYSQL_ROOT_PASSWORD: root。
• 說明： 密碼透過環境變數管理，而非直接硬編碼 (Hardcode) 在 Java 原始碼中，符合安全開發規範。
• 最小化權限與埠號管控：
• 證據： ports: - "8081:8080"。
• 說明： 僅開放必要的服務埠號，並透過非標準埠號 (8081) 提供服務，減少被掃描攻擊的機率。
• 伺服器端輸入過濾：
• 證據： SecureApp.java 原始碼中的 userInput.replace 邏輯。
• 說明： 在後端接收資料時立即進行清洗，確保存入系統的資料不含惡意代碼。

3. 前端安全設計
• 跨站腳本攻擊防護 (XSS Protection)：
• 證據： 截圖顯示輸入 <script>alert('Hacked')</script> 後，頁面正確顯示為純文字。
• 說明： 前端顯示時已進行 HTML 實體轉義 (Escape)，確保腳本不會被瀏覽器執行。
• 通訊埠掩蔽設計：
• 證據： 瀏覽器網址顯示為 localhost:8081。
• 說明： 隱藏後端真實運行的 8080 埠，透過 Docker 映射增加一層代理防護。
• URL 解碼安全處理：
• 證據： 原始碼中 java.net.URLDecoder.decode。
• 說明： 正確處理前端傳回的編碼字串，防止特殊字元繞過過濾邏輯
