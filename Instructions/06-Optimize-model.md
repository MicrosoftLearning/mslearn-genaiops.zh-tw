---
lab:
  title: 使用綜合資料集最佳化模型
---

## 使用綜合資料集最佳化模型

最佳化生成式 AI 應用程式牽涉到利用資料集來增強模型的效能和可靠性。 透過使用綜合資料，開發人員可以模擬各種可能不存在於真實世界資料的案例和邊緣案例。 此外，評估模型的輸出對於取得高品質且可靠的 AI 應用程式至關重要。 您可以使用 Azure AI 評估 SDK 有效率地管理整個最佳化和評估流程，該 SDK 提供強大的工具和架構來簡化這些工作。

## 案例

假設您想要建置 AI 支援的智慧型指南應用程式，以增強博物館中的訪客體驗。 該應用程式旨在回答有關歷史人物的問題。 若要評估應用程式的回應，您需要建立完整的綜合問答資料集，以涵蓋這些人物及其工作的各個層面。

您已選取 GPT-4 模型來提供生成式答案。 您現在想要將生成內容相關互動的模擬器放在一起，評估不同案例中的 AI 效能。

讓我們首先部署建置此應用程式的必要資源。

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

## 設定本機開發環境

若要快速實驗並反覆執行，您將使用附有 Visual Studio (VS) Code 中的 Python 程式碼的筆記本。 準備好將 VS Code 用於本機構想。

1. 開啟 VS Code 並**複製**下列 Git 存放庫：[https://github.com/MicrosoftLearning/mslearn-genaiops.git](https://github.com/MicrosoftLearning/mslearn-genaiops.git)
1. 將複製品儲存在本機磁碟機上，並在複製之後開啟資料夾。
1. 在 VS Code Explorer（左窗格）中，開啟 [Files/06]**** 資料夾中的筆記本 [06-Optimize-your-model.ipynb]****。
1. 執行筆記本中的所有儲存格。

## 清理

如果您已完成 Azure AI 服務探索，您應該刪除在本練習中建立的資源，以避免產生不必要的 Azure 成本。

1. 返回包含 Azure 入口網站的瀏覽器索引標籤 (或在新的瀏覽器索引標籤中重新開啟 [Azure 入口網站](https://portal.azure.com?azure-portal=true))，並檢視您在其中部署本練習所用資源的資源群組內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。
