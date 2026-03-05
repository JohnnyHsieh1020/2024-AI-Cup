# Financial Report Retrieval using LLM

Implementation of a document retrieval pipeline for a financial report QA competition on the [T-Brain platform](https://tbrain.trendmicro.com.tw/Competitions/Details/37).

The goal of the task is not to answer the question directly, but to identify which document contains the answer.

## Problem Description

In the competition setting, each question is associated with:

- A natural language query
- A list of candidate document IDs

**The task is to predict which document ID contains the answer.**

### Example

- Question: 某公司的總資產是多少？

- Possible documents:
  - 112\. A公司合併現金流量表
  - 328\. A公司合併資產負債表
  - 794\. A公司財務報告附註

- Correct reasoning:
總資產 → 出現在資產負債表
→ 選擇 328

## Dataset Structure

The competition dataset contains three document categories:

- FAQ documents
- Insurance policy documents
- Financial reports of listed companies

Each question includes:

| Field    | Description            |
| -------- | ---------------------- |
| qid      | question ID            |
| query    | question text          |
| source   | candidate document IDs |
| category | document category      |

The task is to output:

```json
{
    "qid": question_id,
    "retrieve": predicted_document_id
}
```

**⚠️ The dataset and any derived content are not included in this repository due to competition rules.**

## Method Overview

This project focuses on the financial report category, where questions typically ask about financial statement values.

Example:

1. 營業利益是多少？
2. 總資產是多少？
3. 本期淨利是多少？

These values are usually located in specific financial statements.  
Therefore, identifying the **type of financial document** is an effective strategy for retrieving the correct source document.

## System Pipeline

```mermaid
flowchart LR

A[Question] --> B[Candidate Document IDs]
B --> C[Document Metadata]
C --> D[Prompt Construction]
D --> E[LLM Reasoning]
E --> F[Predicted Document ID]
```

Instead of directly parsing the financial reports, the system uses metadata and reasoning to determine the most likely document.

## Document Tagging

To facilitate document retrieval, each candidate PDF page is associated with a document tag describing the type of financial information contained in the page.

The tagging categories include:

- Balance Sheet
- Income Statement
- Cash Flow Statement
- Statement of Changes in Equity
- Financial Report Notes
- Auditor Review Report
- LLM-generated Summary (used when a page cannot be clearly categorized)

For most documents, tags are **manually assigned** based on the financial report structure.
If the PDF content cannot be clearly classified into predefined categories, an LLM-generated summary is used as the tag to provide additional semantic information.

These tags serve as structured metadata, allowing the LLM to reason about which document is most likely to contain the answer.

## Prompt Design

The system uses two types of prompts in the pipeline:

1. Document Retrieval Prompt – used to determine which candidate document contains the answer.
2. Summary Generation Prompt – used when a document cannot be clearly categorized.

### Document Retrieval Prompt

```Plain Text
你是一位專業的財報專家，請根據以下說明完成任務。
---------------------
options格式範例:
格式: id. [文件說明]
範例:
1.A公司合併資產負債表
2.B公司合併綜合損益表
3.C合併財務報告附註
---------------------
流程範例:
1. question: A公司的營業利益是多少？
2. 提取關鍵字(會計、財務相關專有名次詞): 營業利益
3. 關鍵字與options比對: 因為"營業利益"可以在合併綜合損益表找到，因此選擇 "2.B公司合併綜合損益表"
4. 輸出(只要輸出id): 2
---------------------
注意事項:
1. 只要輸出文件編號，不要說明理由。
2. 輸出文字長度：最多四位數字
3. 若判斷不出來，輸出<option>中最後一個文件的id。
---------------------
<question>{question}</question>
<options>{options}</options>
```

### Summary Generation Prompt

Some document pages cannot be clearly classified into predefined financial statement categories.
For these cases, an LLM-generated summary is used as a metadata tag.

The summary provides a short description of the page content and helps the LLM reason about its relevance.

```Plain Text
你是一位專業的財報專家，擅長提取財報相關文件中重要資訊以及作重點整理，請用淺顯易懂的方式回應。
請根據<content>的內文提取重要資訊並在<summary>內做重點整理。

輸出限制:
1. 只能使用繁體中文進行回覆。
2. 請使用純文字格式輸出。
3. 輸出文字長度：100字

<content>{content}</content>
<summary></summary>
```

## Limitations

- Document tags rely on manual annotation.
- Some pages cannot be clearly categorized.
- LLM reasoning may fail when financial terminology is ambiguous.

## Future improvements

- Automatic document type classification
- Embedding-based candidate filtering
- Hybrid retrieval (metadata + semantic search)
