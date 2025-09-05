---
lab:
  title: 協調 RAG 系統
  description: 了解如何在應用程式中實作擷取擴增生成 (RAG) 系統，以強化所產生回應的正確性和相關性。
---

## 協調 RAG 系統

擷取擴增生成 (RAG) 系統結合了大型語言模型的強大功能與有效率的擷取機制，以增強所生成回應的正確性和相關性。 藉由利用 LangChain 進行協調流程和 Azure AI Foundry 的 AI 功能，我們可以建立強固的管線，從資料集擷取相關信息並生成連貫的回應。 在此練習中，您將逐步設定環境、前置處理資料、建立內嵌，以及建立索引，最終讓您有效地實作 RAG 系統。

此練習大約需要 **30** 分鐘。

## 案例

假設您想要建置一個應用程式，提供關於倫敦酒店的建議。 在應用程式中，您想要代理程式不僅能推薦旅館，還能回答使用者可能針對旅館提出的問題。

您已選取 GPT-4 模型來提供生成式答案。 您現在想要建立一個 RAG 系統，根據其他使用者的評論，為模型提供基礎資料，引導聊天提供個人化的建議。

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
    cd ~/mslearn-genaiops/Files/04/
     ```

1. 輸入下列命令來啟動虛擬環境，並安裝您需要的程式庫：

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv langchain-text-splitters langchain-community langchain-openai
    ```

1. 輸入以下命令，開啟已提供的設定檔：

    ```powershell
   code .env
    ```

    程式碼編輯器中會開啟檔案。

1. 在程式碼檔案中 以您稍早複製的端點和索引鍵值取代 **your_azure_openai_service_endpoint** 和 **your_azure_openai_service_api_key** 預留位置。
1. *取代預留位置後*，在程式碼編輯器中使用 **CTRL+S** 命令或**按下滑鼠右鍵 > [儲存]** 來儲存變更，然後使用 **CTRL+Q** 命令或**按下滑鼠右鍵 > [結束]** 來關閉程式碼編輯器，同時保持 Cloud Shell 命令列開啟。

## 實作 RAG

您現在會執行指令碼來內嵌和預先處理資料、建立內嵌，以及建置向量儲存和索引，最後讓您可以有效實作 RAG 系統。

1. 執行下列命令以**編輯已提供的指令碼**：

    ```powershell
   code RAG.py
    ```

1. 在指令碼中，尋找 **# 初始化要用來從 LangChain 整合套件使用的元件**。 在此註解下方，貼上下列代碼：

    ```python
   # Initialize the components that will be used from LangChain's suite of integrations
   llm = AzureChatOpenAI(azure_deployment=llm_name)
   embeddings = AzureOpenAIEmbeddings(azure_deployment=embeddings_name)
   vector_store = InMemoryVectorStore(embeddings)
    ```

1. 檢閱指令碼，並注意該指令碼會使用.csv 檔案，檔案以酒店評論作為基礎資料。 您可以在命令列窗格中執行命令 `download app_hotel_reviews.csv` 並開啟此檔案，查看檔案中的內容。
1. 接下來，尋找 **# 將文件分割成區塊，以用於內嵌和向量儲存**。 在此註解下方，貼上下列代碼：

    ```python
   # Split the documents into chunks for embedding and vector storage
   text_splitter = RecursiveCharacterTextSplitter(
       chunk_size=200,
       chunk_overlap=20,
       add_start_index=True,
   )
   all_splits = text_splitter.split_documents(docs)
    
   print(f"Split documents into {len(all_splits)} sub-documents.")
    ```

    上述程式碼會將一組大型文件分割成較小的區塊。 這很重要，因為許多內嵌模型 (例如用於語意搜尋或向量資料庫的模型) 都有權杖限制，在較短的文字上執行效果較佳。

1. 接下來，尋找 **# 內嵌每個文字區塊的內容，並將這些內容插入向量存放區**。 在此註解下方，貼上下列代碼：

    ```python
   # Embed the contents of each text chunk and insert these embeddings into a vector store
   document_ids = vector_store.add_documents(documents=all_splits)
    ```

1. 接下來，尋找 **# 依據使用者輸入，從向量存放區抓取相關文件**。 在此註解下方貼上下列程式碼，觀察正確的識別：

    ```python
   # Retrieve relevant documents from the vector store based on user input
   retrieved_docs = vector_store.similarity_search(question, k=10)
   docs_content = "\n\n".join(doc.page_content for doc in retrieved_docs)
    ```

    上述程式碼會搜尋向量存放區中，與使用者輸入問題最為相近的文件。 系統會使用與文件相同的嵌入模型，將問題轉換成向量。 接著，系統會比較此向量與所有已儲存的向量，並抓取最相近的向量。

1. 儲存您的變更。
1. 在命令列中輸入下列命令，以**執行指令碼**：

    ```powershell
   python RAG.py
    ```

1. 執行應用程式之後，您可以開始詢問問題 (例如 `Where can I stay in London?`)，並以更具體的查詢進行後續追蹤。

## 推論

在此練習中，您已建置一個典型 RAG 系統及其主要元件。 透過使用您自己的文件來通知模型的回應，您可以在 LLM 制定回應時提供基本資料。 若為企業解決方案，這表示您可以將生成式 AI 限制在企業內容上。

## 清理

如果您已完成 Azure AI 服務探索，您應該刪除在本練習中建立的資源，以避免產生不必要的 Azure 成本。

1. 返回包含 Azure 入口網站的瀏覽器索引標籤 (或在新的瀏覽器索引標籤中重新開啟 [Azure 入口網站](https://portal.azure.com?azure-portal=true))，並檢視您在其中部署本練習所用資源的資源群組內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。
