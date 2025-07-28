---
lab:
  title: 使用綜合資料集最佳化模型
  description: 了解如何建立綜合資料集，並用其強化模型的效能和可靠性。
---

## 使用綜合資料集最佳化模型

最佳化生成式 AI 應用程式牽涉到利用資料集來增強模型的效能和可靠性。 透過使用綜合資料，開發人員可以模擬各種可能不存在於真實世界資料的案例和邊緣案例。 此外，評估模型的輸出對於取得高品質且可靠的 AI 應用程式至關重要。 您可以使用 Azure AI 評估 SDK 有效率地管理整個最佳化和評估流程，該 SDK 提供強大的工具和架構來簡化這些工作。

此練習大約需要 **30** 分鐘\*

> \* 此估計值不包含練習結束時的選用工作。
## 案例

假設您想要建置 AI 支援的智慧型指南應用程式，以增強博物館中的訪客體驗。 該應用程式旨在回答有關歷史人物的問題。 若要評估應用程式的回應，您需要建立完整的綜合問答資料集，以涵蓋這些人物及其工作的各個層面。

您已選取 GPT-4 模型來提供生成式答案。 您現在想要將生成內容相關互動的模擬器放在一起，評估不同案例中的 AI 效能。

讓我們首先部署建置此應用程式的必要資源。

## 建立 Azure AI 中樞與專案

您可以透過 Azure AI Foundry 入口網站手動建立 Azure AI 中樞和專案，以及部署練習中使用的模型。 不過，您也可以透過使用範本應用程式搭配 [Azure Developer CLI (azd)](https://aka.ms/azd) 來自動化此流程。

1. 在網頁瀏覽器中，在 `https://portal.azure.com` 開啟 [Azure 入口網站](https://portal.azure.com)，並使用您的 Azure 認證登入。

1. 使用頁面頂部搜尋欄右側的 **[\>_]** 按鈕在 Azure 入口網站中建立一個新的 Cloud Shell，並選擇 ***PowerShell*** 環境。 Cloud Shell 會在 Azure 入口網站底部的窗格顯示命令列介面。 如需使用 Azure Cloud Shell 的詳細資訊，請參閱 [Azure Cloud Shell 文件](https://docs.microsoft.com/azure/cloud-shell/overview)。

    > **注意**：如果您之前建立了使用 *Bash* 環境的 Cloud Shell，請將其切換到 ***PowerShell***。

1. 在 Cloud Shell 工具列中，在 [設定]**** 功能表中，選擇 [移至傳統版本]****。

    **<font color="red">繼續之前，請先確定您已切換成 Cloud Shell 傳統版本。</font>**

1. 在 PowerShell 窗格中，輸入以下命令來複製此練習的存放庫：

    ```powershell
   rm -r mslearn-genaiops -f
   git clone https://github.com/MicrosoftLearning/mslearn-genaiops
    ```

1. 複製存放庫之後，請輸入下列命令來初始化入門範本。 
   
    ```powershell
   cd ./mslearn-genaiops/Starter
   azd init
    ```

1. 出現提示后，請為新環境指定名稱，因為它將作為為所有佈建資源提供唯一名稱的基礎。
        
1. 接下來，輸入下列命令以執行入門範本。 它會佈建具有相依資源的 AI 中樞、AI 專案、AI 服務和線上端點。 它也會部署 GPT-4 Turbo、GPT-4o 和 GPT-4o mini 模型。

    ```powershell
   azd up  
    ```

1. 出現提示時，請選擇您想要使用的訂用帳戶，然後選擇下列其中一個位置來進行資源佈建：
   - 美國東部
   - 美國東部 2
   - 美國中北部
   - 美國中南部
   - 瑞典中部
   - 美國西部
   - 美國西部 3
    
1. 等候指令碼完成 - 這通常需要大約 10 分鐘的時間，但在某些情況下，需要更長時間。

    > **注意**：Azure OpenAI 資源在租用戶等級受到區域配額的限制。 上面列出的區域包括本練習中使用的模型類型的預設配額。 隨機選擇區域可以降低單一區域達到其配額限制的風險。 如果達到配額限制，您可能需要在不同的區域中建立另一個資源群組。 深入了解[每個區域的模型可用性。](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models?tabs=standard%2Cstandard-chat-completions#global-standard-model-availability)

    <details>
      <summary><b>疑難排解提示</b>：指定區域中沒有可用的配額</summary>
        <p>如果您因為所選取區域中沒有可用的配額而收到任何模型的部署錯誤，請嘗試執行下列命令：</p>
        <ul>
          <pre><code>azd env set AZURE_ENV_NAME new_env_name
   azd env set AZURE_RESOURCE_GROUP new_rg_name
   azd env set AZURE_LOCATION new_location
   azd up</code></pre>
        將 <code>new_env_name</code>、<code>new_rg_name</code> 和 <code>new_location</code> 替換為新的值。 新位置必須是練習開頭所列的區域之一，例如 <code>eastus2</code>、<code>northcentralus</code> 等。
        </ul>
    </details>

1. 佈建所有資源之後，請使用下列命令來擷取端點和 AI 服務資源的存取金鑰。 請注意，您必須以您的資源群組和 AI 服務資源名稱取代 `<rg-env_name>` 和 `<aoai-xxxxxxxxxx>`。 這兩者都會列印在部署的輸出中。

     ```powershell
    Get-AzCognitiveServicesAccount -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property endpoint
     ```

     ```powershell
    Get-AzCognitiveServicesAccountKey -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property Key1
     ```

1. 複製這些值，因為稍後會使用這些值。

## 在 Cloud Shell 中設定您的開發環境

若要快速實驗並逐一查看，您需要在 Cloud Shell 中使用一組 Python 指令碼。

1. 在 Cloud Shell 命令行窗格中輸入下列命令，以瀏覽至資料夾，資料夾中包含此練習中使用的程式碼檔案：

     ```powershell
    cd ~/mslearn-genaiops/Files/06/
     ```

1. 輸入下列命令來啟動虛擬環境，並安裝您需要的程式庫：

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv azure-ai-evaluation azure-ai-projects promptflow wikipedia aiohttp openai==1.77.0
    ```

1. 輸入以下命令，開啟已提供的設定檔：

    ```powershell
   code .env
    ```

    程式碼編輯器中會開啟檔案。

1. 在程式碼檔案中 以您稍早複製的端點和索引鍵值取代 **your_azure_openai_service_endpoint** 和 **your_azure_openai_service_api_key** 預留位置。
1. *取代預留位置後*，在程式碼編輯器中使用 **CTRL+S** 命令或**按下滑鼠右鍵 > [儲存]** 來儲存變更，然後使用 **CTRL+Q** 命令或**按下滑鼠右鍵 > [結束]** 來關閉程式碼編輯器，同時保持 Cloud Shell 命令列開啟。

## 產生綜合資料

請執行產生綜合資料集的程式碼，並用其評估預先訓練模型的品質。

1. 執行下列命令以**編輯已提供的指令碼**：

    ```powershell
   code generate_synth_data.py
    ```

1. 在指令碼中，尋找 **# Define callback function**。
1. 在此註解下方，貼上下列代碼：

    ```
    async def callback(
        messages: List[Dict],
        stream: bool = False,
        session_state: Any = None,  # noqa: ANN401
        context: Optional[Dict[str, Any]] = None,
    ) -> dict:
        messages_list = messages["messages"]
        # Get the last message
        latest_message = messages_list[-1]
        query = latest_message["content"]
        context = text
        # Call your endpoint or AI application here
        current_dir = os.getcwd()
        prompty_path = os.path.join(current_dir, "application.prompty")
        _flow = load_flow(source=prompty_path)
        response = _flow(query=query, context=context, conversation_history=messages_list)
        # Format the response to follow the OpenAI chat protocol
        formatted_response = {
            "content": response,
            "role": "assistant",
            "context": context,
        }
        messages["messages"].append(formatted_response)
        return {
            "messages": messages["messages"],
            "stream": stream,
            "session_state": session_state,
            "context": context
        }
    ```

    您可以指定目標回呼函數，將任何應用程式端點帶入模擬。 在此情況下，請使用具備 Prompty 檔案 `application.prompty` 的 LLM 應用程式。 上述回呼函數會執行下列工作，以處理模擬器所產生的每個訊息：
    * 擷取最新的使用者訊息。
    * 從 application.prompty 載入提示流程。
    * 使用提示流程生成回應。
    * 格式化回應以遵守 OpenAI 聊天通訊協定。
    * 將助理的回應附加至訊息清單。

    >**注意**：如需使用 Prompty 的詳細資訊，請參閱 [Prompty 的使用者文件](https://www.prompty.ai/docs)。

1. 接下來，尋找 **# Run the simulator**。
1. 在此註解下方，貼上下列代碼：

    ```
    model_config = {
        "azure_endpoint": os.getenv("AZURE_OPENAI_ENDPOINT"),
        "api_key": os.getenv("AZURE_OPENAI_API_KEY"),
        "azure_deployment": os.getenv("AZURE_OPENAI_DEPLOYMENT"),
    }
    
    simulator = Simulator(model_config=model_config)
    
    outputs = asyncio.run(simulator(
        target=callback,
        text=text,
        num_queries=1,  # Minimal number of queries
    ))
    
    output_file = "simulation_output.jsonl"
    with open(output_file, "w") as file:
        for output in outputs:
            file.write(output.to_eval_qr_json_lines())
    ```

   上述程式碼會初始化並執行模擬器，以根據先前從維基百科擷取的文字產生綜合對話。

1. 接下來，尋找 **# Evaluate the model**。
1. 在此註解下方，貼上下列代碼：

    ```
    groundedness_evaluator = GroundednessEvaluator(model_config=model_config)
    eval_output = evaluate(
        data=output_file,
        evaluators={
            "groundedness": groundedness_evaluator
        },
        output_path="groundedness_eval_output.json"
    )
    ```

    現在您已擁有資料集，就可以評估生成式 AI 應用程式的品質和有效性。 在上述程式碼中，請使用基礎性作為品質計量。

1. 儲存您的變更。
1. 在程式碼編輯器下的 Cloud Shell 命令列窗格中，輸入下列命令來**執行指令碼**：

    ```
   python generate_synth_data.py
    ```

    指令碼完成後，您可以執行 `download simulation_output.jsonl` 與 `download groundedness_eval_output.json`，檢閱其內容，再下載輸出檔案。 如果基礎計量並未接近 3.0，您可以變更 `application.prompty` 檔案中的 LLM 參數 (例如 `temperature`、`top_p`、`presence_penalty` 或 `frequency_penalty`)，然後重新執行指令碼來產生新的資料集以供評估。 您也可以變更 `wiki_search_term`，以根據不同的內容取得綜合資料集。

## (選用) 微調模型

如果您還有時間，您可以使用產生的資料集，在 Azure AI Foundry 中微調模型。 微調取決於雲端基礎結構資源。視資料中心容量和並行需求而定，佈建該資源可能需要一段時間。

1. 開啟新的瀏覽器索引標籤，然後瀏覽至位於 `https://ai.azure.com` 的 [Azure AI Foundry 入口網站](https://ai.azure.com)，使用您的 Azure 認證登入。
1. 在 AI Foundry 的首頁中，選取您在練習開始時建立的專案。
1. 使用左側功能表瀏覽至 [建置和自訂]**** 區段下的 [微調]**** 頁面。
1. 選取按鈕以新增微調模型、選取 **gpt-4o** 模型，然後選取 [下一步]****。
1. 使用下列設定**微調**模型：
    - **模型版本**：*選取預設版本*
    - **自訂方法**：監督式
    - **模型尾碼**：`ft-travel`
    - **已連線的 AI 資源**：*選取您在建立中樞網站時所建立的連線。預設應選取。*
    - **訓練資料**：上傳檔案

    <details>  
    <summary><b>疑難排解提示</b>：權限錯誤</summary>
    <p>如果您收到權限錯誤，請嘗試下列方法進行疑難排解：</p>
    <ul>
        <li>在 Azure 入口網站選取 AI 服務資源。</li>
        <li>在 [資源管理] 下的 [身分識別] 索引標籤中，確認其為系統所指派的受控識別。</li>
        <li>瀏覽至相關聯的儲存體帳戶。 在 [IAM] 頁面上新增角色指派<em>儲存體 Blob 資料擁有者</em>。</li>
        <li>在 [經存取權指派給]<strong></strong> 下，選擇 [受控識別]<strong></strong>、[選取成員]<strong>+</strong>，並選取 [所有系統指派的受控識別]<strong></strong>，然後選取您的 Azure AI 服務資源。</li>
        <li>檢閱並指派以儲存新的設定，然後重試上一個步驟。</li>
    </ul>
    </details>

    - **上傳檔案**：選取您在上一個步驟下載的 JSONL 檔案。
    - **驗證資料**：無
    - **工作參數**：*保留預設設定*
1. 微調將會開始，不過可能需要一些時間才能完成。

    > **注意**：微調和部署可能需要較長時間 (30 分鐘或更長)，因此您可能需要定期檢查。 到目前為止，您可以選取微調模型工作，並檢視其 [記錄]**** 索引標籤，以查看進度的詳細資料。

## (選用) 部署微調的模型

微調成功完成時，您可以部署微調的模型。

1. 選取微調工作連結，以開啟其詳細資料頁面。 然後選取 [計量]**** 索引標籤，探索微調計量。
1. 使用下列設定部署微調的模型：
    - **部署名稱**：*模型部署的有效名稱*
    - **部署類型**：標準
    - **每分鐘權杖速率限制 (千)**：5K * (如果您的訂用帳戶少於 5K，則為可用上限)
    - **內容篩選**：預設
1. 請等候部署完成，才能進行測試。這可能需要一些時間。 在成功之前請檢查**佈建狀態** (可能需要重新整理瀏覽器，以查看更新狀態)。
1. 部署準備就緒時，瀏覽至微調的模型，並選取 [在遊樂場中開啟]****。

    現在您已部署微調的模型，您可以在 Chat 遊樂場中進行測試，就像使用任何基本模型一樣。

## 推論

在此練習中，您建立了綜合資料集，以模擬使用者與聊天完成應用程式之間的交談。 使用此資料集，您可以評估應用程式回應的品質並加以微調，以達到所需的結果。

## 清理

如果您已完成 Azure AI 服務探索，您應該刪除在本練習中建立的資源，以避免產生不必要的 Azure 成本。

1. 返回包含 Azure 入口網站的瀏覽器索引標籤 (或在新的瀏覽器索引標籤中重新開啟 [Azure 入口網站](https://portal.azure.com?azure-portal=true))，並檢視您在其中部署本練習所用資源的資源群組內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。
