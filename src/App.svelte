<script>
  let error = null;
  let token = "";
  let alphavantageApiKey = "";
  let message = "";
  let answer = "";
  let processOnGoing = false;
  let messages = [];

  const onSubmit = async () => {
    processOnGoing = true;
    // 初期化
    answer = "";
    error = null;
    messages = [
      {
        role: "system",
        content: "入力に対し、現在の株価を取得して返信してください。",
      },
    ];

    messages.push({
      role: "user",
      content: message,
    });
    let response;
    try {
      response = await fetchChatCompletionsApi(messages);
    } catch (err) {
      error = err;
    }

    answer = response;

    processOnGoing = false;
  };

  const fetchChatCompletionsApi = async (messages) => {
    const response = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({
        model: "gpt-4",
        messages: messages,
        functions: [
          {
            name: "getStockData",
            description: "株価を取得する",
            parameters: {
              type: "object",
              properties: {
                ticker: {
                  type: "string",
                  description: "銘柄コード",
                },
              },
            },
          },
        ],
      }),
    });

    if (!response.ok) {
      const {
        error: { message },
      } = await response.json();

      error = message;
      return;
    }

    const { choices } = await response.json();

    const finish_reason = choices[0].finish_reason;

    switch (finish_reason) {
      case "stop":
        return choices[0].message.content;
      case "function_call":
        await handleFunctionCall(choices[0]);
        return await fetchChatCompletionsApi(messages);
      default:
        error = "エラーが発生しました。";
    }
  };

  const handleFunctionCall = async (choice) => {
    const {
      function_call: { name, arguments: arg },
    } = choice.message;
    if (name !== "getStockData") {
      error = "関数呼び出しに失敗しました。";
      return;
    }
    const { ticker } = JSON.parse(arg);
    let stockData;
    try {
      stockData = await getStockData(ticker);
    } catch (err) {
      error = err;
    }

    messages.push({
      role: "function",
      name: "getStockData",
      content: JSON.stringify(stockData),
    });
  };

  const getStockData = async (ticker) => {
    const response = await fetch(
      `https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol=${ticker}&interval=1min&apikey=${alphavantageApiKey}`,
      {
        method: "GET",
      },
    );

    if (!response.ok) {
      throw new Error(await response.json());
    }

    return await response.json();
  };
</script>

<main>
  <form on:submit|preventDefault={onSubmit}>
    <label>
      alphavantageAPIのトークンを貼り付けてください。
      <input type="text" id="token" bind:value={alphavantageApiKey} />
    </label>
    <label>
      OpenAI APIのトークンを貼り付けてください。
      <input type="text" id="token" bind:value={token} />
    </label>
    <label>
      株価を取得したいアメリカの会社を入力してください。
      <textarea id="textarea" bind:value={message}></textarea>
    </label>
    <button
      type="submit"
      disabled={processOnGoing || message === "" || token === ""}
      >{processOnGoing ? "処理中" : "送信"}</button
    >
  </form>

  {#if error}
    <div class="error">{error}</div>
  {:else}
    <div>{@html answer.replace(/\n/g, "<br>")}</div>
  {/if}
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    gap: 48px;
  }
  label {
    display: flex;
    align-items: center;
  }
  textarea {
    width: 300px;
    height: 100px;
  }
  button {
    width: 200px;
  }
  form {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;

    gap: 24px;
  }
  .error {
    color: red;
  }
</style>
