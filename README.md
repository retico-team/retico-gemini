# retico-gemini
A ReTiCo module that supports the Google Gemini API. It accepts both text and audio as input and streams the model's response as text. This module was initially written for [this network](https://github.com/mi-1000/retico-language-practice-network) and then adapted for general purpose.

## Installation and Requirements

To use the Gemini module you first need to install the retico-gemini package:
* Install retico-gemini:
```pip install retico-gemini```

  or, for the latest development version from GitHub:
```pip install git+https://github.com/retico-team/retico-gemini.git```

### API key
The module needs a Google API key in the `GOOGLE_API_KEY` environment variable. The simplest way is to create a `.env` file in the folder you run your script from:
```dotenv
GOOGLE_API_KEY=your-api-key
```
The API key is loaded automatically. You can get an API key for free on the [Google Cloud website](https://cloud.google.com/vertex-ai/generative-ai/docs/start/api-keys).

> [!NOTE]
> A single API key should work for both the Google Gemini and Google ASR APIs, but in case not, follow the setup tutorial for the Google ASR Retico module [here](https://github.com/retico-team/retico-googleasr).

## Modules

### `GeminiModule`
Subscribes to `TextIU` or `AudioIU` and outputs `TextIU`. Text is sent to Gemini when it is committed, or after `timeout_interval` seconds without new text; audio is sent once the speaker stops talking. The response tokens are streamed forward as they are generated.

The module can keep the conversation in the model's context window, and can change the system instructions dynamically with a callback function.

#### Arguments:
* `system_instructions` (str): the system instructions given to the model to guide its responses; defaults to `"You are a helpful assistant."`
* `model` (str): the Gemini model to use (see the [list of models](https://ai.google.dev/gemini-api/docs/models)); defaults to `"gemini-2.5-flash"`
* `temperature` (float): the temperature of the model, usually from `0.0` (more deterministic) to `2.0` (more creative); defaults to `1.0`
* `top_p` (float): influences how the model selects output tokens; defaults to `1.0`
* `seed` (int or `"random"`): the seed for token generation; an integer makes the generation deterministic; defaults to `"random"`
* `max_output_tokens` (int): the maximum number of tokens the model may generate; defaults to `65535`
* `thinking_budget` (int): the maximum number of tokens the model may use for thinking, if it supports thinking; defaults to `2048`
* `contents` (list[`google.genai.types.Content`]): messages to start the conversation with; defaults to `None` (empty conversation)
* `keep_context` (bool): if `True`, previous turns are kept as context; if `False`, each turn is independent; defaults to `True`
* `timeout_interval` (float): seconds without new text after which the current text is sent to the model; defaults to `1.5`
* `update_instructions_callback` (callable): called with `(iu, update_type)` for every incoming IU; return a string to replace the system instructions, or `None` to keep them; defaults to `None`

### Example
**Note:** The example uses `MicrophoneModule` and `CallbackModule` from `retico_core` and `GoogleASRModule` from [retico-googleasr](https://github.com/retico-team/retico-googleasr) (`pip install retico-googleasr`). It only works with English input as is, due to limitations with `GoogleASRModule`. To send audio to the API instead of text, subscribe the Gemini module directly to the microphone module and remove the ASR module.

```python
from retico_core import network
from retico_core.audio import MicrophoneModule
from retico_core.debug import CallbackModule
from retico_googleasr import GoogleASRModule
from retico_gemini import GeminiModule

def callback(um):
    for iu, ut in um:
        print(f"{ut}: {iu.text}")

def set_instructions(iu, ut):
    if hasattr(iu, "text") and "France" in iu.text:
        return "You are now a French-speaking assistant. Finish all of your utterances with 'Vive la France!'"

if __name__ == "__main__":
    mic = MicrophoneModule(rate=16000)
    asr = GoogleASRModule(rate=16000)
    llm = GeminiModule("You are a knowledgeable assistant holding a conversation with the user. Talk as if you were a noble scientist from the Renaissance.", update_instructions_callback=set_instructions)
    debug = CallbackModule(callback)

    mic.subscribe(asr)
    asr.subscribe(llm)
    llm.subscribe(debug)

    network.run(mic)
    input("Running...\n")
    network.stop(mic)
```
