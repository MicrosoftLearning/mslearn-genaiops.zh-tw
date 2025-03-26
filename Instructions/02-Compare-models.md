---
lab:
  title: 從模型目錄選取語言模型。
---

## 從模型目錄選取語言模型。

當定義使用案例時，您就可以使用模型目錄，探索 AI 模型是否可以解決您的問題。 您可以使用模型目錄，選擇想部署的模型，然後就可以比較模型，以便探索哪個模型最能滿足您的需求。

在這份練習中，您可以透過 Azure AI Foundry 入口網站中的模型目錄，比較兩種語言模型。

此練習大約需要 **25** 分鐘。

## 案例

假設您想要建置應用程式，以協助學生瞭解如何在 Python 中編寫程式碼。 在應用程式中，您想要自動化導師，以協助學生編寫和評估程式碼。 在某項練習中，學生必須根據以下範例圖片，編寫繪製圓形圖所需的 Python 程式碼：

![圓形圖顯示考試成績，其中分數學 (34.9%)、物理 (28.6%)、化學 (20.6%) 和英語 (15.9%) 四個部分](./images/demo.png)

您必須選取可以接受圖片作為輸入，也能產生準確的程式碼語言模型。 符合這些準則的可用模型有 GPT-4 Turbo、GPT-4o 和 GPT-4o mini。

讓我們先從 Azure AI Foundry 入口網站中，部署使用這些模型的必要資源。

## 建立 Azure AI 中樞與專案

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
    Get-AzCognitiveServicesAccountKey -ResourceGroupName <rg-env_name> -Name <aoai-xxxxxxxxxx> | Select-Object -Property Key1
     ```

1. 複製這些值，因為稍後會使用這些值。

## 比較模型

您知道，有三種接受影像作為輸入的模型，據推斷，基礎結構完全受控於 Azure。 目前您必須比較看看，才能確定哪一種最適合我們的使用案例。

1. 請在網頁瀏覽器中，開啟 [Azure AI Foundry 入口網站](https://ai.azure.com)，再於`https://ai.azure.com` 時使用您的 Azure 認證完成登入。
1. 如果出現提示，請選取之前建立的 AI 專案。
1. 使用左側的功能表，瀏覽至 **[模型目錄]** 頁面。
1. 選取 **[比較模型]**（請在搜尋窗格中，尋找篩選條件旁的按鈕）。
1. 移除指定模型。
1. 逐一新增您想要比較的三種模型：**gpt-4**、**gpt-4o** 和 **gpt-4o-mini**。 針對 **gpt-4**，請確定指定版本是 **turbo-2024-04-09**，原因是此為唯一接受影像作為輸入的版本。
1. 請將 X 軸變更為 **[正確性]**。
1. 請確定已將 Y 軸設定為 **[成本]**。

檢閱繪圖，嘗試回答下列問題：

- *哪種模型比較正確？*
- *哪種模型比較便宜？*

主要會根據公開可用的通用資料集，計算基準衡量標準的正確性。 我們已經可以從圖中篩選掉其中某一個模型，原因是每種權杖的成本最高，正確性卻不一定最高。 做出決定之前，先讓我們先探討一下，怎樣針對您的使用案例，列出剩餘兩種模型的輸出品質。

## 設定本機開發環境

若要快速實驗並反覆執行，您將使用附有 Visual Studio (VS) Code 中的 Python 程式碼的筆記本。 準備好將 VS Code 用於本機構想。

1. 開啟 VS Code 並**複製**下列 Git 存放庫：[https://github.com/MicrosoftLearning/mslearn-genaiops.git](https://github.com/MicrosoftLearning/mslearn-genaiops.git)
1. 將複製品儲存在本機磁碟機上，並在複製之後開啟資料夾。
1. 請在 VS Code Explorer（左側窗格）中，開啟 **02-Compare-models.ipynb** 中的筆記本，存放在 **Files/02** 資料夾中。
1. 執行筆記本中的所有儲存格。

## 清理

如果您已完成 Azure AI 服務探索，您應該刪除在本練習中建立的資源，以避免產生不必要的 Azure 成本。

1. 返回包含 Azure 入口網站的瀏覽器索引標籤 (或在新的瀏覽器索引標籤中重新開啟 [Azure 入口網站](https://portal.azure.com?azure-portal=true))，並檢視您在其中部署本練習所用資源的資源群組內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。
