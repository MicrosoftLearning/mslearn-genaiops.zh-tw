---
lab:
  title: 使用 Prompty 探索提示工程
  description: 了解如何使用 Prompty，以透過您的語言模型快速測試及改善不同的提示，並確保其經過建構及協調，可獲得最佳結果。
---

## 使用 Prompty 探索提示工程

此練習大約需要 **45 分鐘**。

> **注意**：本練習假設您對 Azure AI Foundry 有一定的瞭解，因此有些說明刻意不那麼詳盡，以鼓勵更積極地探索和實踐學習。

## 簡介

在構想期間，您想要使用語言模型快速測試並改善不同的提示。 您可以透過各種方式進行提示功能，包括透過 Azure AI Foundry 入口網站中的遊樂場，或使用 Prompty 以取得更加程式碼優先的方法。

在此練習中，您會使用透過 Azure AI Foundry 部署的模型，在 Azure Cloud Shell 中使用 Prompty 探索提示工程。

## 設定環境

若要完成本練習中的工作，您需要：

- Azure AI Foundry 中樞；
- Azure AI Foundry 專案；
- 已部署的模型 (例如 GPT-4o)。

### 建立 Azure AI 中樞與專案

> **注意**：如果您已經有 Azure AI 中樞和專案，您可以跳過此程序，並使用現有的專案。

您可以透過 Azure AI Foundry 入口網站手動建立 Azure AI 專案，以及部署練習中使用的模型。 不過，您也可以透過使用範本應用程式搭配 [Azure Developer CLI (azd)](https://aka.ms/azd) 來自動化此流程。

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

### 在 Cloud Shell 中設定您的虛擬環境

若要快速實驗並逐一查看，您需要在 Cloud Shell 中使用一組 Python 指令碼。

1. 在 Cloud Shell 命令行窗格中輸入下列命令，以瀏覽至資料夾，資料夾中包含此練習中使用的程式碼檔案：

     ```powershell
    cd ~/mslearn-genaiops/Files/03/
     ```

1. 輸入下列命令來啟動虛擬環境，並安裝您需要的程式庫：

    ```powershell
   python -m venv labenv
   ./labenv/bin/Activate.ps1
   pip install python-dotenv openai tiktoken azure-ai-projects prompty[azure]
    ```

1. 輸入以下命令，開啟已提供的設定檔：

    ```powershell
   code .env
    ```

    程式碼編輯器中會開啟檔案。

1. 在程式碼檔案中，以您稍早複製的端點和金鑰值取代 **ENDPOINTNAME** 和 **APIKEY** 預留位置。
1. *取代預留位置後*，在程式碼編輯器中使用 **CTRL+S** 命令或**按下滑鼠右鍵 > [儲存]** 來儲存變更，然後使用 **CTRL+Q** 命令或**按下滑鼠右鍵 > [結束]** 來關閉程式碼編輯器，同時保持 Cloud Shell 命令列開啟。

## 將系統提示最佳化

將系統提示的長度減到最小同時維持生成式 AI 中的功能，是大規模部署的基礎。 若提示較短，回應時間就會比較快，因為 AI 模型需要處理的權杖數與使用的運算資源都比較少。

1. 請輸入下列命令，開啟已提供的應用程式檔案：

    ```powershell
   code optimize-prompt.py
    ```

    請檢閱程式碼，並注意指令碼會執行已經預先定義系統提示的 `start.prompty` 範本檔案。

1. 執行 `code start.prompty` 以檢閱系統提示。 請考慮如何縮短提示，同時維持其意圖清晰且有效。 例如：

   ```python
   original_prompt = "You are a helpful assistant. Your job is to answer questions and provide information to users in a concise and accurate manner."
   optimized_prompt = "You are a helpful assistant. Answer questions concisely and accurately."
   ```

   移除多餘的文字，並專注於基本指示。 將經過最佳化的提示儲存在檔案中。

### 測試並驗證您的最佳化提示

測試提示變更很重要，因為這可以確保您減少權杖使用量，同時兼顧品質。

1. 執行 `code token-count.py` 以開啟並檢閱練習中提供的權杖計數器應用程式。 如果您使用的最佳化提示與上述範例所提供提示不同，您也可以在此應用程式中使用。

1. 使用 `python token-count.py` 執行指令碼，並觀察權杖計數器的差異。 確保最佳化提示仍會產生高品質回應。

## 分析使用者互動

了解使用者如何與您的應用程式互動，有助於找出權杖使用量變多的模式。

1. 檢閱使用者提示的範例資料集：

    - **「摘要《戰爭與和平》** 的劇情。」**
    - **「有哪些與貓相關的有趣事實？」**
    - **「針對著手使用 AI 最佳化供應鏈，撰寫詳細的商務計畫。」**
    - **「將『嗨，您好』翻譯成法文。」**
    - **「向 10 歲的孩童說明量子糾纏」。**
    - **「給我 10 個科幻短篇故事的創意構想。」**

    分別判斷每個提示可能導致 AI 回應內容**較短**、**中等**或**較長/複雜**。

1. 檢閱您的分類。 您注意到哪些模式？ 請考慮：

    - 提示的**抽象程度** (例如創意與實際間的差異) 會影響回應長度嗎？
    - **開放式提示**是否會出現較長回應？
    - **指示複雜程度** (例如「當我是 10 歲孩童那樣解釋」) 對回應長度會造成什麼影響？

1. 輸入下列命令執行 **optimize-prompt** 應用程式：

    ```
   python optimize-prompt.py
    ```

1. 使用上面提供的一些範例來驗證您的分析。
1. 現在，使用下列較長提示，並檢閱輸出結果：

    ```
   Write a comprehensive overview of the history of artificial intelligence, including key milestones, major contributors, and the evolution of machine learning techniques from the 1950s to today.
    ```

1. 根據以下要求，重寫這個提示：

    - 限制範圍
    - 設定對簡潔度的期待
    - 使用格式或結構來引導回應

1. 比較回應，以驗證您確實得到更簡潔的答案。

> **注意**：您可以使用 `token-count.py` 以比較這兩個回應中的權杖使用量。
<br>
<details>
<summary><b>重寫提示的範例：</b></summary><br>
<p>「提供 AI 歷程記錄中 5 個關鍵里程碑的重點摘要。」</p>
</details>

## [**選用**] 在真實案例中套用您的最佳化提示

1. 假設您正在建置客戶支援聊天機器人，該聊天機器人必須提供快速又準確的答案。
1. 將經過最佳化的系統提示和範本整合到聊天機器人的程式碼 (*您可以使用 `optimize-prompt.py` 做為起點*)。
1. 使用各種使用者查詢測試聊天機器人，以確保機器人能快速有效的提出回應。

## 推論

提示最佳化是降低成本並提升生成式 AI 應用程式效能的關鍵技能。 您可以縮短提示、使用範本及分析使用者互動，以建立更有效率且可擴充的解決方案。

## 清理

如果您已完成 Azure AI 服務探索，您應該刪除在本練習中建立的資源，以避免產生不必要的 Azure 成本。

1. 返回包含 Azure 入口網站的瀏覽器索引標籤 (或在新的瀏覽器索引標籤中重新開啟 [Azure 入口網站](https://portal.azure.com?azure-portal=true))，並檢視您在其中部署本練習所用資源的資源群組內容。
1. 在工具列上，選取 [刪除資源群組]****。
1. 輸入資源群組名稱並確認您想要將其刪除。
