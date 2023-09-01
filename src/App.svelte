<script>
  let error = null;
  let token = "";
  let message = "";
  let answer = "";
  let processOnGoing = false;

  const onSubmit = async () => {
    processOnGoing = true;
    answer = "";
    error = null;
    const response = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({
        model: "gpt-3.5-turbo",
        stream: true,
        frequency_penalty: 1,
        messages: [
          {
            role: "system",
            content:
              "入力された文章をビジネスで使う敬語にして、10通りの選択肢を出力してください。各選択肢の先頭には1. 2. などの番号を振って、改行してください。",
          },
          {
            role: "user",
            content: message,
          },
        ],
      }),
    });

    if (!response.ok) {
      const {
        error: { message },
      } = await response.json();

      error = message;
      processOnGoing = false;
      return;
    }

    const decoder = new TextDecoder();
    const reader = response.body.getReader();

    while (true) {
      const { done, value } = await reader.read();

      const events = decoder
        .decode(value)
        // 複数の`data: `行がまとまって落ちてくることもある
        .split("\n\n")
        .map((line) => {
          try {
            return JSON.parse(line.split("data: ")[1]);
          } catch {
            return null;
          }
        })
        .filter(Boolean);

      for (const json of events) {
        const content = json.choices[0].delta?.content ?? "";

        answer += content;
      }

      if (done) break;
    }
    processOnGoing = false;
  };
</script>

<main>
  <form on:submit|preventDefault={onSubmit}>
    <label>
      OpenAI APIのトークンを貼り付けてください。
      <input type="text" id="token" bind:value={token} />
    </label>
    <label>
      ビジネス敬語に直したい文章を入力してください。
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
