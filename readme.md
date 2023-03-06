# 🛫 BrowserPilot

An intelligent web browsing agent controlled by natural language.

![demo](assets/demo_buffalo.gif)

Language is the most natural interface through which humans give and receive instructions. Instead of writing bespoke automation or scraping code which is brittle to changes, creating and adding agents should be as simple as writing plain English.

## 🏗️ Installation

1. `pip install browserpilot`
2. Download Chromedriver (latest stable release) from [here](https://sites.google.com/chromium.org/driver/) and place it in the same folder as this file. Unzip. In Finder, right click the unpacked chromedriver and click "Open". This will remove the restrictive default permissions and allow Python to access it.
3. Create an environment variable in your favorite manner setting OPENAI_API_KEY to your API key.


## 🦭 Usage
### 🗺️ API
The form factor is fairly simple (see below).

```python
from browserpilot.agents.gpt_selenium_agent import GPTSeleniumAgent

instructions = """Go to Google.com
Find all text boxes.
Find the first visible text box.
Click on the first visible text box.
Type in "buffalo buffalo buffalo buffalo buffalo" and press enter.
Wait 2 seconds.
Find all anchor elements that link to Wikipedia.
Click on the first one.
Wait for 10 seconds."""

agent = GPTSeleniumAgent(instructions, "/path/to/chromedriver")
agent.run()
```

The harder (but funner) part is writing the natural language prompts.


### 📑 Writing Prompts

It helps if you are familiar with how Selenium works and programming in general. This is because this project uses GPT-3 to translate natural language into code, so you should be as precise as you can. In this way, it is more like writing code with Copilot than it is talking to a friend; for instance, it helps to refer to things as text boxes (vs. "search box") or "button which says 'Log in'" rather than "the login button". Sometimes, it will also not pick up on specific words that are important, so it helps to break them out into separate lines. Instead of "find all the visible text boxes", you do "find all the text boxes" and then "find the first visible text box".

You can look at some examples in `prompts/examples` to get started.

Create "functions" by enclosing instructions in `BEGIN_FUNCTION func_name` and `END_FUNCTION`, and then call them by starting a line with `RUN_FUNCTION` or `INJECT_FUNCTION`. Below is an example: 

```
BEGIN_FUNCTION search_buffalo
Go to Google.com
Find all text boxes.
Find the first visible text box.
Click on the first visible text box.
Type in "buffalo buffalo buffalo buffalo buffalo" and press enter.
Wait 2 seconds.
Get all anchors on the page that contain the word "buffalo".
Click on the first link.
END_FUNCTION

RUN_FUNCTION search_buffalo
Wait for 10 seconds.
```

You may also choose to create a yaml or json file with a list of instructions. In general, it needs to have an `instructions` field, and optionally a `compiled` field which has the processed code, as well as an optional `chrome_options` field. For instance, if you were using BrowserPilot to download something and needed to set the default download directory and bypass the dialog box for downloading:

```json
{
    "download.default_directory": "/path/to/download/directory",
    "download.prompt_for_download": False
}
```

See [buffalo wikipedia example](prompts/examples/buffalo_wikipedia.yaml).

You may pass a `instruction_output_file` to the constructor of GPTSeleniumAgent which will output a yaml file with the compiled instructions from GPT-3, to avoid having to pay API costs. 

### 🎬 Using the Studio CLI

The BrowserPilot studio is a CLI that is meant to make it easier to iteratively generate prompts. See `run_studio.py` to see how to run the studio class.

```
  'exit': Exits the Studio.
  'run last': Replay last compiled routine.
  'compile': Compiles the routine.
  'run': Compiles and runs the routine.
  'clear': Clears the routine.
  'help': Shows this message.
  'list': Shows the routine so far.
  'delete': Deletes the last line.
  'save': Saves the routine to a yaml file.
```

The flow could look something like this:
1. Add natural language commands line by line.
2. Run `compile` when you are ready, and it will ask the LLM to translate it into Selenium code.
3. Use `run last` to run that Selenium code (without any additional API calls!) Or simply use `run` to compile AND run.
4. Watch the Selenium browser come up and work its magic! You can eyeball it to see if it works, or see the stack trace printed to console if it doesn't.
5. Use `list` to see the natural language commands so far. Use `delete` to remove the last line of the prompt or `clear` to wipe it entirely. 
6. Finally, when you are done, `save` can save it to yaml or `exit` to simply leave. 

## ✋🏼 Contributing
There are two ways I envision folks contributing.

- **Adding to the Prompt Library**: Read "Writing Prompts" above and simply make a pull request to add something to `prompts/`! At some point, I will figure out a protocol for folder naming conventions and the evaluation of submitted code (for security, accuracy, etc). This would be a particularly attractive option for those who aren't as familiar with coding.
- **Contributing code**: I am happy to take suggestions! The main way to add to the repository is to extend the capabilities of the agent, or to create new agents entirely. The best way to do this is to familiarize yourself with "Architecture and Prompt Patterns" above, and to (a) expand the list of capabilities in the base prompt in `InstructionCompiler` and (b) write the corresponding method in `GPTSeleniumAgent`. 

## ⛩️ Architecture and Prompt Patterns

This repo was inspired by the work of [Yihui He](https://github.com/yihui-he/ActGPT), [Adept.ai](https://adept.ai/), and [Nat Friedman](https://github.com/nat/natbot). In particular, the basic abstractions and prompts used were built off of Yihui's hackathon code. The idea to preprocess HTML and use GPT-3 to intelligently pick elements out is from Nat. 

- The prompts used can be found in [instruction compiler](agents/compilers/instruction_compiler.py). The base prompt describes in plain English a set of actions that the browsing agent can take, some general conventions on how to write code, and some constraints on its behavior. **These actions correspond one-for-one with methods in `GPTSeleniumAgent`**. Those actions, to-date, include:
    - `env.driver`, the Selenium webdriver.
    - `env.find_elements(by='id', value=None)` finds and returns list of elements.
    - `env.find_element(by='id', value=None)` is similar to `env.find_elements()` except it only returns the first element.
    - `env.find_nearest(e, xpath)` can be used to locate an element near another one.
    - `env.send_keys(element, text)` sends `text` to element.
    - `env.get(url)` goes to url.
    - `env.click(element)` clicks the element.
    - `env.wait(seconds)` waits for `seconds` seconds.
    - `env.scroll(direction, iframe=None)` scrolls the page. Will switch to `iframe` if given. `direction` can be "up", "down", "left", or "right". 
    - `env.get_llm_response(text)` asks AI about a string `text`.
    - `env.retrieve_information(prompt, entire_page=False)` returns a string, information from a page given a prompt. Use prompt="Summarize:" for summaries. Uses all the text if entire_page=True and only visible text if False. Invoked with commands like "retrieve", "find in the page", or similar.
    - `env.ask_llm_to_find_element(description)` asks AI to find an element that matches the description.
    - `env.save(text, filename)` saves the string `text` to a file `filename`.
    - `env.get_text_from_page(entire_page)` returns the free text from the page. If entire_page is True, it returns all the text from HTML doc. If False, returns only visible text.
- The rest of the code is basically middleware which exposes a Selenium object to GPT-3. **For each action mentioned in the base prompt, there is a corresponding method in GPTSeleniumAgent.**
    - An `InstructionCompiler` is used to parse user input into semantically cogent blocks of actions.


### 🎉 Finished
- [x] JSON loading.
- [x] Basic iframe support.
- [x] GPTSeleniumAgent should be able to load prompts and cached successful runs in the form of yaml files. InstructionCompiler should be able to save instructions to yaml.
- [x] 💭 Add a summarization capability to the agent.
- [x] Demo/test something where it has to ask the LLM to synthesize something it reads online.
- [x] 🚨 Figured out how to feed the content of the HTML page into the GPT-3 context window and have it reliably pick out specific elements from it, that would be great!


## 🚨 Disclaimer 🚨

This package runs code output from the OpenAI API in Python using `exec`. 🚨 **This is not considered a safe convention** 🚨. Accordingly, you should be extra careful when using this package. The standard disclaimer follows.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

