---
lab:
  title: 使用追蹤來分析和偵錯您的生成式 AI 應用程式
---

# 使用追蹤來分析和偵錯您的生成式 AI 應用程式

本練習大約需要 **30** 分鐘的時間。

> **注意**：本練習假設您對 Azure AI Foundry 有一定的瞭解，因此有些說明刻意不那麼詳盡，以鼓勵更積極地探索和實踐學習。

## 簡介

在此練習中，您將執行多步驟的生成式 AI 助理，為您的徒步旅行提供建議，並建議戶外裝備。 您將使用 Azure AI 推斷 SDK 的追蹤功能來分析應用程式執行方式，並識別模型和周圍邏輯所做出的關鍵決策點。

您將與已部署的模型互動，模擬真實的使用者旅程圖、追蹤應用程式的每個階段，從使用者輸入到模型回應到後續處理，并檢視 Azure AI Foundry 中的追蹤資料。 這可協助您理解追蹤如何增強可檢視性、簡化偵錯，並支援生成式 AI 應用程式的效能最佳化。

## 設定環境

若要完成本練習中的工作，您需要：

- Azure AI Foundry 中樞；
- Azure AI Foundry 專案；
- 已部署的模型（例如 GPT-4o）；
- 已連線的 Application Insights 資源。

### 建立 AI Foundry 中樞與專案

若要快速設定中樞和專案，以下提供使用 Azure AI Foundry 入口網站 UI 的簡單指示。

1. 瀏覽至 Azure AI Foundry 入口網站：開啟 [https://ai.azure.com](https://ai.azure.com)。
1. 使用您的 Azure 認證進行登入。
1. 建立專案：
    1. 瀏覽至 [所有中樞 + 專案]****。
    1. 選取 [+ 新增專案]****。
    1. 輸入**專案名稱**。
    1. 出現提示時， **請建立新的中樞**。
    1. 自訂中樞：
        1. 選取適當的 [訂用帳戶]****、[資源群組]** **和 [位置] 等****。
        1. 連結**新的 Azure AI 服務**資源（跳過 AI 搜尋）。
    1. 檢閱並選取 [建立]****。
1. **請等候部署完成**（大約 1-2 分鐘）。

### 部署模型

若要產生您可以監視的資料，您必須先部署模型並與其互動。 指示會要求您部署 GPT-4o 模型，但**您可以使用 Azure OpenAI 服務集合中可供您使用的任何模型**。

1. 使用左側的功能表，在 [我的資產]**** 中，選取 [模型 + 端點]**** 頁面。
1. 部署**基本模型**，然後選擇 [gpt-4o]****。
1. **自訂部署詳細資料**。
1. 將 [容量]**** 設定為 [每分鐘 5K 個權杖 (TPM)]****。

中樞和專案已就緒，並會自動佈建所有必要的 Azure 資源。

### 連接 Application Insights

將 Application Insights 連線到 Azure AI Foundry 中的專案，開始收集用於分析的資料。

1. 在 Azure AI Foundry 入口網站中打開您的專案。
1. 使用左側的功能表，然後選取 [追蹤]**** 頁面。
1. **建立新的** Application Insights 資源以連線到您的應用程式。
1. 輸入 **Application Insights 資源名稱**。

Application Insights 現在已連線到您的專案，並開始收集資料以進行分析。

## 使用 Cloud Shell 執行生成式 AI 應用程式

您將從 Azure Cloud Shell 連線到 Azure AI Foundry 專案，並作為生成式 AI 應用程式的一部分，以程式設計方式與已部署的模型互動。

### 與已部署的模型互動

首先，擷取為了與您的模型互動而要驗證的必要資訊。 然後，您將存取 Azure Cloud Shell，並更新您的生成式 AI 應用程式的代碼。

1. 在 Azure AI Foundry 入口網站中，檢視專案的**概觀**頁面。
1. 在 [專案詳細資料]**** 區域中，記下 [專案連接字串]****。
1. 將字串**儲存**在記事本中。 您將使用此連接字串連線到用戶端應用程式中的專案。
1. 開啟一個新的瀏覽器索引標籤（保持 Azure AI Foundry 入口網站在現有索引標籤中開啟）。
1. 在新索引標籤中，瀏覽到 `https://portal.azure.com` 的 [Azure 入口網站](https://portal.azure.com)；如果出現提示，請使用您的 Azure 認證登入。
1. 使用頁面頂部搜尋欄右側的 **[\>_]** 按鈕在 Azure 入口網站中建立一個新的 Cloud Shell，並選擇 ***PowerShell 環境*** (訂用帳戶中沒有儲存體)。
1. 在 Cloud Shell 工具列中，在 [設定]**** 功能表中，選擇 [移至傳統版本]****。

    **<font color="red">繼續之前，請先確定您已切換成 Cloud Shell 傳統版本。</font>**

1. 在 Cloud Shell 窗格中，輸入並執行下列命令：

    ```
    rm -r mslearn-genaiops -f
    git clone https://github.com/microsoftlearning/mslearn-genaiops mslearn-genaiops
    ```

    此命令會複製 GitHub 存放庫，其中包含此練習的程式碼檔案。

1. 複製存放庫之後，瀏覽至包含應用程式碼檔案的資料夾：  

    ```
   cd mslearn-genaiops/Files/08
    ```

1. 在 Cloud Shell 命令列窗格中，輸入下列命令來安裝您需要的程式庫：

    ```
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv azure-identity azure-ai-projects azure-ai-inference azure-monitor-opentelemetry
    ```

1. 輸入以下命令，開啟已提供的設定檔：

    ```
   code .env
    ```

    程式碼編輯器中會開啟檔案。

1. 在程式碼檔案中：

    1. 將 **your_project_connection_string** 預留位置替換為專案的連接字串（從 Azure AI Foundry 入口網站中的專案**概觀**頁面複製）。
    1. 將 **your_model_deployment** 預留位置替換為您指派給 GPT-4o 模型部署的名稱（預設為 `gpt-4o`）。

1. 替換預留位置*之後*，請在程式碼編輯器中使用 **CTRL+S** 命令或**按右鍵 > [儲存]** 來**儲存變更**。

### 更新您的生成式 AI 應用程式的代碼

現在您的環境已設定，且已設定 .env 檔案，現在可以準備 AI 助理指令碼來執行。 在與 AI 專案連線並啟用 Application Insights 旁邊，您需要：

- 與已部署的模型互動
- 定義函式以指定您的提示。
- 定義呼叫所有函式的主要流程。

您會將這三個部分新增至起始指令碼。

1. 執行下列命令以**開啟已提供的指令碼**：

    ```
   code start-prompt.py
    ```

    您會看到數個索引鍵行已保留空白字元或標示為空白 # 註解。 您的工作是將下列正確行複製並貼到適當的位置，以完成指令碼。

1. 在指令碼中，尋找 **#函式来呼叫模型并控制代碼追蹤**。
1. 在此註解下方，貼上下列代碼：

    ```
   def call_model(system_prompt, user_prompt, span_name):
        with tracer.start_as_current_span(span_name) as span:
            span.set_attribute("session.id", SESSION_ID)
            span.set_attribute("prompt.user", user_prompt)
            start_time = time.time()
    
            response = chat_client.complete(
                model=model_name,
                messages=[SystemMessage(system_prompt), UserMessage(user_prompt)]
            )
    
            duration = time.time() - start_time
            output = response.choices[0].message.content
            span.set_attribute("response.time", duration)
            span.set_attribute("response.tokens", len(output.split()))
            return output
    ```

1. 在指令碼中，尋找 **# 函式，以根據使用者喜好設定**推薦徒步。
1. 在此註解下方，貼上下列代碼：

    ```
   def recommend_hike(preferences):
        with tracer.start_as_current_span("recommend_hike") as span:
            prompt = f"""
            Recommend a named hiking trail based on the following user preferences.
            Provide only the name of the trail and a one-sentence summary.
            Preferences: {preferences}
            """
            response = call_model(
                "You are an expert hiking trail recommender.",
                prompt,
                "recommend_model_call"
            )
            span.set_attribute("hike_recommendation", response.strip())
            return response.strip()
    ```

1. 在指令碼中，找出 **#---- Main Flow ----**。
1. 在此註解下方，貼上下列代碼：

    ```
   if __name__ == "__main__":
       with tracer.start_as_current_span("trail_guide_session") as session_span:
           session_span.set_attribute("session.id", SESSION_ID)
           print("\n--- Trail Guide AI Assistant ---")
           preferences = input("Tell me what kind of hike you're looking for (location, difficulty, scenery):\n> ")

           hike = recommend_hike(preferences)
           print(f"\n✅ Recommended Hike: {hike}")

           # Run profile function


           # Run match product function


           print(f"\n🔍 Trace ID available in Application Insights for session: {SESSION_ID}")
    ```

1. **儲存您在指令碼中所做的變更**。
1. 在程式碼編輯器下的 Cloud Shell 命令列窗格中，輸入下列命令來**執行指令碼**：

    ```
   python start-prompt.py
    ```

1. 提供您要尋找之徒步旅行類型的一些描述，例如：

    ```
   A one-day hike in the mountains
    ```

    模型會產生回應，這會使用 Application Insights 擷取以進一步分析。 您可以在** Azure AI Foundry 入口網站中**進行視覺化追蹤。

> **注意**：監視資料可能需要幾分鐘的時間才會顯示在 Azure 監視器中。

## 在 Azure AI Foundry 入口網站中檢視追蹤資料

執行指令碼後，您擷取了 AI 應用程式的執行追蹤。 現在您將使用 Azure AI Foundry 中的 Application Insights 來探索。

> **備註：** 稍後，您將再次執行代碼，並在 Azure AI Foundry 入口網站中再次檢視追蹤。 讓我們先探索在哪裡尋找視覺化追蹤。

### 瀏覽至 Azure AI Foundry 入口網站

1. **讓 Cloud Shell 保持開啟！** 您將返回此代碼，以更新代碼並再次執行。
1. 在開啟 **Azure AI Foundry 入口網站**的情況下，瀏覽至瀏覽器中的索引標籤。
1. 使用左側功能表，選取 [追蹤]****。
1. *如果* 未顯示任何資料，請 **重新整理** 您的檢視。
1. 選取追蹤 **train_guide_session** 開啟顯示更多詳細資料的新視窗。

### 檢閱您的追蹤。

此檢視表會顯示「軌跡指南 AI 助理」完整會話的跡數。

- **最上層範圍**：trail_guide_session。這是父範圍。 它代表您助理將全部執行，從開始到完成。

- **巢狀子範圍**：每個縮排行都代表巢狀作業。 您會發現：

    - **建議_徒步**，它捕捉到你的邏輯來決定徒步旅行。
    - ** recommend_model_call**，這是 recommend_hike 內 call_model（） 所建立的跨度。
    - **chat gpt-4o** ，由 Azure AI 推斷 SDK 自動檢測，以顯示實際的 LLM 互動。

1. 您可以按一下任何範圍來檢視：

    1. 時間長度。
    1. 其屬性，例如使用者提示、使用權杖、回應時間。
    1. 附加** span.set_attribute（...）** 的任何錯誤或自定義資料。

## 將更多函式新增至您的代碼


1. 輸入下列命令**重新開啟此指令碼**：

    ```
   code start-prompt.py
    ```

1. 在指令碼中，尋找 **# 函式為建議的徒步旅行產生旅行個人檔案**。
1. 在此註解下方，貼上下列代碼：

    ```
   def generate_trip_profile(hike_name):
       with tracer.start_as_current_span("trip_profile_generation") as span:
           prompt = f"""
           Hike: {hike_name}
           Respond ONLY with a valid JSON object and nothing else.
           Do not include any intro text, commentary, or markdown formatting.
           Format: {{ "trailType": ..., "typicalWeather": ..., "recommendedGear": [ ... ] }}
           """
           response = call_model(
               "You are an AI assistant that returns structured hiking trip data in JSON format.",
               prompt,
               "trip_profile_model_call"
           )
           print("🔍 Raw model response:", response)
           try:
               profile = json.loads(response)
               span.set_attribute("profile.success", True)
               return profile
           except json.JSONDecodeError as e:
               print("❌ JSON decode error:", e)
               span.set_attribute("profile.success", False)
               return {}
    ```

1. 在指令碼中，尋找 **# 函式，以符合目錄中產品的推薦項目齒輪圖**。
1. 在此註解下方，貼上下列代碼：

    ```
   def match_products(recommended_gear):
       with tracer.start_as_current_span("product_matching") as span:
           matched = []
           for gear_item in recommended_gear:
               for product in mock_product_catalog:
                   if any(word in product.lower() for word in gear_item.lower().split()):
                       matched.append(product)
                       break
           span.set_attribute("matched.count", len(matched))
           return matched
    ```

1. 在指令碼中，找出 **#執行設定檔函式**。
1. 在下方**對齊**此註解，貼上下列代碼：

    ```
           profile = generate_trip_profile(hike)
           if not profile:
           print("Failed to generate trip profile. Please check Application Insights for trace.")
           exit(1)

           print(f"\n📋 Trip Profile for {hike}:")
           print(json.dumps(profile, indent=2))
    ```

1. 在指令碼中，尋找 **#執行符合產品函式**。
1. 在下方**對齊**此註解，貼上下列代碼：

    ```
           matched = match_products(profile.get("recommendedGear", []))
           print("\n🛒 Recommended Products from Lakeshore Retail:")
           print("\n".join(matched))
    ```

1. **儲存您在指令碼中所做的變更**。
1. 在程式碼編輯器下的 Cloud Shell 命令列窗格中，輸入下列命令來**執行指令碼**：

    ```
   python start-prompt.py
    ```

1. 提供您要尋找之徒步旅行類型的一些描述，例如：

    ```
   I want to go for a multi-day adventure along the beach
    ```

> **注意**：監視資料可能需要幾分鐘的時間才會顯示在 Azure 監視器中。

### 在 Azure AI Foundry 入口網站中檢視新的追蹤

1. 瀏覽回 Azure AI Foundry 入口網站。
1. 應該會出現名稱相同 **trail_guide_session** 的新追蹤。 如有必要，請重新整理檢視。
1. 選取新的追蹤以開啟更詳細的檢視。
1. 檢閱新的巢狀子範圍 **trip_profile_generation** 和 **product_matching**。
1. 選取 **product_matching** 並檢閱出現的中繼資料。

    在 product_matching 函式中，包含**span.set_attribute（“matched.count”， len（matched））。** 藉由機碼值組**matched.count** 和變數相符的長度來設定 屬性，即可將這項資訊新增至 **product_matching** 追蹤。 您可以在中繼資料中的**屬性**找到這個機碼值組。

## （選用）追蹤錯誤

如果您有額外的時間，您可以檢閱在發生錯誤時如何使用追蹤。 可能會向您提供擲回的錯誤指令碼。 執行它並檢閱追蹤。

這是一項旨在挑戰您的練習，這意味著說明故意不太詳細。

1. 在 Cloud Shell 中，開啟 **error-prompt.py** 指令碼。 此指令碼位於與** start-prompt.py **指令碼相同的目錄中。 檢閱內容。
1. 執行** error-prompt.py** 指令碼。 出現提示時，在命令行中提供答案。
1. *希望*，輸出訊息包含**無法產生旅程設定檔。請檢查 Application Insights 以取得追蹤。**。
1. 瀏覽至** trip_profile_generation**追蹤，並檢查發生錯誤的原因。

<br>
<details>
<summary><b>取得答案</b>：為什麼您遇到錯誤...</summary><br>
<p>如果您檢查 generate_trip_profile 函式的 LLM 追蹤，您會發現助理回應包含倒引號和 JSON 一詞，以將輸出格式化為程式碼區塊。

雖然這有助於顯示，但它會導致代碼問題，因為輸出已不再為有效 JSON。 這會導致進一步處理期間發生剖析錯誤。

此錯誤可能是由 LLM 指示遵循其輸出的特定格式所造成。 在使用者提示中包含指示會比在系統提示中更有效。</p>
</details>

## 哪裡可以找到其他實驗室

您可以在 [Azure AI Foundry 學習入口網站](https://ai.azure.com)中探索其他實驗和練習，或參閱課程的**實驗部分**瞭解其他可用的活動。
