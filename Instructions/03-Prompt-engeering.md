---
lab:
  title: 使用 Prompty 探索提示工程
---

## 使用 Prompty 探索提示工程

在構想期間，您想要使用語言模型快速測試並改善不同的提示。 您可以透過各種方式進行提示功能，包括透過 Azure AI Foundry 入口網站中的遊樂場，或使用 Prompty 以取得更加程式碼優先的方法。

在此練習中，您會使用透過 Azure AI Foundry 部署的模型，在 Visual Studio Code 中使用 Prompty 探索提示工程。

此練習大約需要 **40** 分鐘。

## 案例

假設您想要建置應用程式，以協助學生瞭解如何在 Python 中編寫程式碼。 在應用程式中，您想要自動化導師，以協助學生編寫和評估程式碼。 不過，您不希望聊天應用程式只提供所有答案。 您希望學生收到個人化提示，鼓勵他們思考如何繼續。

您已選取要開始實驗的 GPT-4 模型。 您現在想要套用提示工程，以引導聊天成為生成個人化提示的導師。

讓我們首先在 Azure AI Foundry 入口網站中部署使用此模型的必要資源。

## 建立 Azure AI 中樞與專案

> **注意**：如果您已經有 Azure AI 中樞和專案，您可以略過此程序，並使用現有的專案。

您可以透過 Azure AI Foundry 入口網站手動建立 Azure AI 中樞和專案，以及部署練習中使用的模型。 不過，您也可以透過使用範本應用程式搭配 [Azure Developer CLI (azd)](https://aka.ms/azd) 來自動化此流程。

1. 在網頁瀏覽器中，在 `https://portal.azure.com` 開啟 [Azure 入口網站](https://portal.azure.com)，並使用您的 Azure 認證登入。

1. 使用頁面頂部搜尋欄右側的 **[\>_]** 按鈕在 Azure 入口網站中建立一個新的 Cloud Shell，並選擇 ***PowerShell*** 環境。 Cloud Shell 會在 Azure 入口網站底部的窗格顯示命令列介面。 如需使用 Azure Cloud Shell 的詳細資訊，請參閱 [Azure Cloud Shell 文件](https://docs.microsoft.com/azure/cloud-shell/overview)。

    > **注意**：如果您之前建立了使用 *Bash* 環境的 Cloud Shell，請將其切換到 ***PowerShell***。

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
        
1. 接下來，輸入下列命令以執行入門範本。 它會佈建具有相依資源的 AI 中樞、AI 專案、AI 服務和線上端點。

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
    Get-AzCognitiveServicesAccountKey -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property Key1
     ```

1. 複製這些值，因為稍後會使用這些值。
   
## 設定本機開發環境

若想快速實驗並逐一查看，您將可在 Visual Studio (VS) Code 中，使用 Prompty。 準備好將 VS Code 用於本機構想。

1. 開啟 VS Code 並**複製**下列 Git 存放庫：[https://github.com/MicrosoftLearning/mslearn-genaiops.git](https://github.com/MicrosoftLearning/mslearn-genaiops.git)
1. 將複製品儲存在本機磁碟機上，並在複製之後開啟資料夾。
1. 請在 VS Code 的 [擴充] 窗格中，搜尋並安裝 **Prompty** 擴充功能。
1. 請在 VS Code Explorer（左邊窗格）中，用滑鼠按右鍵，點兩下 [檔案/03]**** 資料夾。
1. 請在下拉式功能表中，選取** [新的 Prompty]**。
1. 開啟剛才建立的檔案，檔名為 **[基本 Prompty]**。
1. 選取右上角的 **[播放]** 按鈕，（或可按下 F5 鍵），以便執行 Prompty 檔案。
1. 出現提示時，登入伺服器，選取 **[允許]**。
1. 選取 Azure 帳戶，然後登入。
1. 返回 VS Code，會開啟其中的**輸出**窗格，然後顯示錯誤訊息。 錯誤訊息應該告訴您還未指定，或是找不到部署的型號。

若想修正錯誤，您就必須為 Prompty 設定好想使用的型號。

## 更新提示的中繼資料

若想執行 Prompty 檔案，您就必須指定語言模型，可用來產生回覆。 中繼資料可定義為 Prompty 檔案的 *frontmatter*。 讓我們使用模型設定和其他資訊，更新中繼資料。

1. 開啟 Visual Studio Code 終端機窗格。
1. 複製 **basic.prompty** 檔案 （請在相同資料夾中），再將複本重新命名為 `chat-1.prompty`。
1. 開啟 **chat-1.prompty**，然後更新下列欄位，即可變更某些基本資訊：

    - **Name**：

        ```yaml
        name: Python Tutor Prompt
        ```

    - **描述**：

        ```yaml
        description: A teaching assistant for students wanting to learn how to write and edit Python code.
        ```

    - **已部署的模型**：

        ```yaml
        azure_deployment: ${env:AZURE_OPENAI_CHAT_DEPLOYMENT}
        ```

1. 接下來，請在 **azure_deployment**參數下，預留位置新增 API 金鑰的以下預留位置。

    - **端點金鑰**：

        ```yaml
        api_key: ${env:AZURE_OPENAI_API_KEY}
        ```

1. 儲存更新的 Prompty 檔案。

Prompty 檔案現在具有所有必要的參數，但有些參數使用預留位置，方便取得所需的數值。 會將預留位置儲存在相同資料夾中的 **.env**檔案中。

## 更新模型設定

若想指定 Prompty 使用的模型，您就必須在 .env 檔案中，提供型號資訊。

1. 開啟 **.env** 檔案，只要到 **Files/03** 資料夾即可完成。
1. 使用您之前從 Azure 入口網站中的模型部署輸出那邊複製到的數值，更新每個預留位置：

    ```yaml
    - AZURE_OPENAI_CHAT_DEPLOYMENT="gpt-4"
    - AZURE_OPENAI_ENDPOINT="<Your endpoint target URI>"
    - AZURE_OPENAI_API_KEY="<Your endpoint key>"
    ```

1. 儲存 .env 檔案。
1. 重新執行 **chat-1.prompty** 檔案。

您目前應該可以取得 AI 產生的回應，儘管這類回應只是使用樣本輸入，與您的案例無關。 讓我們先更新範本，把範本變成 AI 助教。

## 編輯樣本區段

樣本區段會指定 Prompty 的輸入，會在還沒提供任何輸入時，先提供想要使用的預設值。

1. 編輯下列參數的欄位：

    - **firstName**：選擇任何其他名稱。
    - **context**：移除整個章節。
    - **question**：用以下提供的文字來取代：

    ```yaml
    What is the difference between 'for' loops and 'while' loops?
    ```

    目前**樣本**章節看起來應該如下所示：
    
    ```yaml
    sample:
    firstName: Daniel
    question: What is the difference between 'for' loops and 'while' loops?
    ```

    1. 執行更新的 Prompty 檔案，同時查看輸出。

