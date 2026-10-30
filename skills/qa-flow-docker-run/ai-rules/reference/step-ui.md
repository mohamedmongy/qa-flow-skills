# QA Flow — Step Type: `ui` (run a UI automation script)

A `ui` step runs **the user's own script** against **one app of a scheme**. The step says, in this order, what it targets (a **scheme** and the one **app** of it the step drives — the app brings its platform), which **engine** the script drives, which **language** it is written in, and then holds the script — so the flow map shows what runs a step without opening its script. The script is standalone: it opens its session with QA Flow's session helpers (`open_page()` / `open_driver()` from `qa_flow_session` — see *Opening the session*), drives the app or the website, and exits. **One script runs on every target** (Local, LambdaTest, BrowserStack): the run's target decides where the helper connects, so the script never names a hub, an endpoint or a provider. QA Flow does not pick commands or map them to an engine — it fills in the flow's placeholders, starts the script the way its language is run, and reads the exit code and the values the script exports.

What happens to a command is the script's business. A Python script sends a command to Appium, Appium routes it to the XCUITest (iOS) or UiAutomator2 (Android) driver, and the driver performs it on the simulator, emulator or device. A script for another engine or language works the same way once it is listed in `Common/ui/runners.py`.

There are no action lists, no `target`, no `runner`, no `cloud`, no step-level `runs_on` / `executor` / `device`, no `release` / `platform` and no flow-level `ui` / `ui.targets` block: a flow carrying them is refused on save. **Where a step runs is the flow's, not the step's** — the flow's target for the scheme (its `app_targets` pin, else the scheme's `default_target`) binds the step's app to a provider, a device and an optional executor; a scheme without targets can't run ui steps (see *The target*). The **Maestro engine was removed**: a step with `engine: "maestro"` is refused on save and fails at run with *the Maestro engine was removed — rewrite the step as an Appium script*. **The release is not the step's** — it is picked when the flow runs (see *The target*).

## The `ui` step

```json
{
  "type": "ui",
  "name": "request_otp",
  "scheme": "monty",
  "app": "merchant_ios",
  "engine": "appium",
  "language": "python",
  "timeout": 300,
  "script": "from appium import webdriver  # noqa: F401\nfrom qa_flow_session import open_driver\n\nwith open_driver(caps={\n    \"platformName\": \"iOS\", \"appium:automationName\": \"XCUITest\",\n    \"appium:app\": \"{{release.app}}\", \"appium:noReset\": True,\n}) as driver:\n    driver.find_element(\"accessibility id\", \"Log in\").click()\n"
}
```

| Field | Values | Notes |
|---|---|---|
| `scheme` | a scheme's key | required — the apps blueprint (Environments → Schemes & releases; `list_app_releases`) |
| `app` | an app key of that scheme | required — the one app this step drives (e.g. `merchant_ios`, `seller_ios`, `portal`); its platform (`ios` / `android` / `web`) comes with it |
| `engine` | `appium`, `playwright` | required — what the script drives: Appium for a mobile app, Playwright (or Appium through WebDriver) for a website |
| `language` | `python` | required — what the script is written in; must be one of the engine's languages. `python` runs on the project's interpreter, so the project's Appium / Playwright clients import |
| `script` | text | required — the whole script, as the user gave it |
| `options` | object | optional — typed options, `{"<name>": value}`. Each option is declared once (`catalog` -> `options`: name, kind — `string`, `integer`, `number`, `boolean`, `choice` or `object` —, choices and the engines / languages / platforms / providers it applies to). Validated on save (known name, in scope, value of its kind) and handed to the script as the env var `QA_FLOW_OPTION_<NAME>` holding the value as JSON. The catalog lists none today, so any name is refused. |
| `timeout` | seconds | optional; default 600 (`QA_FLOW_UI_STEP_TIMEOUT` changes the default for a run) |
| `description` | text | optional |

## The target: schemes, releases, runners

A **scheme** lists the **apps** a product ships, each a key, a name and one platform — `ios`, `android` or `web`; a platform may carry several apps (an iOS Merchant app and an iOS Seller app) (`list_app_releases`; `manage_scheme`). An **app release** of a scheme gives each of its apps a value (`manage_app_release`) — **builds only**:

- **iOS / Android app** — an uploaded **build**: the key of an `.ipa` / `.apk` in Assets → Mobile builds (`upload_asset` gives every build a key and reads its app id and version from the file). Optional `app_id` overrides the build's bundle id / package. A new or edited release never takes provider `refs`; only the migration's `legacy` release still carries them (read-only).
- **Web app** — the website `url` (`{{env.*}}` allowed).

One release of each scheme is its **default** (`default_release`; a scheme's first release becomes it; `manage_scheme` `default_release` or `manage_app_release` `make_default` moves it). **A run picks the release, not the step:** the dashboard's Run buttons (flow editor, flows list, test group) ask for one release per scheme the flows' ui steps use, starting on the default; MCP `run_test_suite` / `execute_test_group` take `app_releases: {"<scheme>": "<release>"}`; the runner takes `--app-release SCHEME=RELEASE` (repeatable) or a plan file's `app_releases:`; anything not picked uses the scheme's default (the subprocess sees the picks as `QA_FLOW_RELEASES`). A scheme with no default and no pick fails its ui steps, naming the scheme.

**Target profiles — where each app runs.** A scheme may also hold named **targets** (`targets`, with a `default_target`): per app of the scheme, a **binding** `{provider, device: {name?, os_version?, kind: simulator|emulator|real}, executor?, caps?}` — e.g. `mixed` = iOS on BrowserStack, Android local. A device has only `name`, `os_version` and `kind` (for a web app `name` is the desktop OS, e.g. `Windows 11` / `MacOS Tahoe` on LambdaTest, `Windows` + `os_version` `11` on BrowserStack); anything else the provider should get goes in the binding's optional `caps` object, checked against **that provider's profile only** — another provider's vendor key or a key the profile doesn't allow is refused on save with the allowed list. They are kept in the scheme (Environments → Schemes & releases; `manage_scheme` `targets` / `default_target`; `targets: []` clears them). A `local` binding (the built-in Local provider only) may also set `install`: `"changed"` (the default — the app is reinstalled only when the release's build changed, keeping its data) or `"fresh"` (reinstalled every run, clearing its data); the target editor shows it as *App install* on Local bindings, any other value is refused on save, and cloud bindings ignore it (clouds install the uploaded build each session). A ui step takes its provider, device and executor from the chosen target's binding for its app — the step itself never names them (`runs_on` / `executor` / `device` on a step are refused on save, by the dashboard and the MCP safeguard). **A scheme without targets can't run ui steps:** the flow editor shows *This scheme has no targets yet — add one in Environments → Schemes & releases* and the save is refused until one is added. **A run picks the target** like it picks a release: the Run picker's target select (starting on the flow's own target — *This flow's target*, or *Each flow's own target* for a group run); MCP `app_targets: {"<scheme>": "<target>"}` (or per app, see *A flow pins its target*) on `run_test_suite` / `execute_test_group`; the runner's `--app-target SCHEME=TARGET` or, for one app, `--app-target SCHEME.APP=TARGET` (repeatable; `--app-target legacy.android=local-android --app-target legacy=cloud-ios-2` = `{"legacy": {"android": "local-android", "*": "cloud-ios-2"}}`, which has `"*"` and so replaces the flow's pin for `legacy`; `SCHEME.APP=TARGET` flags alone override only those apps); a plan file's `app_targets:` (both forms); the subprocess sees the picks as `QA_FLOW_TARGETS`. Nothing picked uses `default_target`.

**A flow pins its target** with a top-level `app_targets` on the flow (MCP `create_flow` / `update_flow` `app_targets`; left out on an update keeps the stored pin, `{}` clears it). Per scheme the pick is either **a target** — `{"<scheme>": "<target>"}`: every app of the scheme runs there, and an app it doesn't bind is an error (it does not fall through) — or **a target per app** — `{"<scheme>": {"<app>": "<target>", "*": "<target>"}}`: each key is an app of the scheme and its target must bind it; the optional `"*"` covers the scheme's other apps (it must be a target of the scheme); an app with neither falls through to the next layer. So one flow can run step 1's iOS app on TestMu, step 2's Android app locally and step 3's website on TestMu without a combined target. Per (scheme, app) the target is: the run's pick, else the flow's pin, else `default_target` — within a pick the app's own key beats `"*"`. Layers resolve one by one, per scheme: an upper pick that is a string, or an object **with** `"*"`, replaces the lower layers' pick for that scheme entirely (its `"*"` covers every app it doesn't name — the lower layer's app keys don't survive); an upper object **without** `"*"` overrides only the apps it names (the lower layer's other entries, its `"*"` included, stay). A sub-flow runs under its parent's pin the same way (the parent is the upper layer). The pin names keys only — the provider, device and executor stay on the scheme's target, so steps carry no provider. In the flow editor the pin is the **🎯 Targets** control (one row per scheme the flow's ui steps use; *Scheme default* = no pin), and each ui step card shows a read-only line *Runs on: the flow's target — <target name>* that follows it. A pin to a scheme without targets, or to a target the scheme doesn't have, a per-app key that isn't an app of the scheme, or a per-app target that doesn't bind that app is refused on save (naming the app and the target) and fails the step at run time; so is a target (pinned or default) that doesn't bind the step's app. The flow's save checks (`validate_flow` too — it uses the stored pin when the request leaves `app_targets` out) run against the pinned target's provider for the step's app. A pinned target can't be removed from its scheme, nor re-keyed (`manage_scheme` answers 409 naming the flows).

**A group can run on several targets at once (matrix run).** Instead of one `app_targets` pick, a group run may take `targets: [{"<scheme>": <pick>}, …]` (each pick in either form) (MCP `execute_test_group` `targets`; the Run dialog's *➕ Add another target set*; a plan's `Group("name", targets=[…])`). Two or more entries run the whole group once per entry, the passes in parallel (up to 4 at a time): every target is preflighted before any pass starts (each problem prefixed with its target, e.g. `[monty=local] …`); `stop_on_failure` stops that target's pass only, and a failed target never cancels the others. Each item result carries `target` (the pick map as `scheme=target, …`; a per-app pick reads `legacy=android:local-android,ios:cloud-ios-2`), the group result carries `targets: [{target, status, passed, failed}]`, and the group passes only if every target passed (a target stopped by `stop_on_failure` has not). One entry is a plain `app_targets` run; sending both `app_targets` and `targets`, or the same target set twice, is refused (400). In a plan, each entry merges over the plan's `app_targets:` (the entry wins per scheme by the same layer rule — an object entry without `"*"` overrides only the apps it names). Each flow item runs in its own process with its env, so parallel targets never share environment variables. A matrix run on cloud targets spends provider minutes per target — confirm it like any run.

**Every run is checked before it starts.** A flow / suite run (▶ Run, `run_test_suite` / `run_case` / `run_multiple_suites`), a test group run (once for the whole group, before its first item), the runner (`docker/user_runner.py`, a `--plan`) resolve the target of every ui step they will execute — the flows named and the sub-flows they call (flow steps and a conditional's `run_flow` targets), each under the run's pick, else its pin (a sub-flow under its caller's), else `default_target` — and check it with the save checks — first the run's own target picks (the `--app-target` flags, `QA_FLOW_TARGETS`, a plan's or a group's target sets, the request's `app_targets`) get the pin's checks for every entry, used by a step or not: an app key the scheme doesn't have, or a target that doesn't exist or doesn't bind the app it is picked for, refuses the run (`the run's app_targets: …`) — then each step (a scheme without targets, a target that doesn't bind the app, a broken binding, a script naming a provider, a known-broken combination, the step's options, a removed engine), the binding's `caps` for the step's exact session, and the release the run picks. Any problem refuses the whole run before anything started: the dashboard routes answer `400 {"success": false, "error": "<summary>", "problems": ["<flow> › step <n> '<name>' (target '<key>'): <problem>", …]}` (the Run dialog and the group view list them; the MCP run tools return the 400 with its problems), the runner prints them and exits `2`. A run without ui steps is not slowed. There is no step-level fallback: a ui step on a scheme without targets fails with *This scheme has no targets yet — add one in Environments → Schemes & releases*.

**Provider credentials belong to the environment, not the release** — Environments → edit → *Device cloud providers* (`BROWSERSTACK_USERNAME` / `BROWSERSTACK_ACCESS_KEY`, `LT_USERNAME` / `LT_ACCESS_KEY`, `APPIUM_URL`), i.e. that environment's variables (`ai-rules/negotiation/env-vars.md`).

Restoring a flow's migration backup (`backups/…` of `scripts/migrate_ui_to_targets.py`) does not restore `data/app_releases.json` — restore the store's own backup `backups/app_releases_<stamp>.json` by hand.

Schemes and releases are the same in every environment. An app a flow step targets can't be removed from its scheme or change platform; a scheme flow steps target can't be deleted; the default release can't be deleted while the scheme has others; a build a release names can't be deleted from Assets.

**What may run where:**

| Platform | Engine | provider a target may bind it to |
|---|---|---|
| web | playwright, appium | local, browserstack, lambdatest |
| ios / android | appium | local, browserstack, lambdatest |

An engine that is not in this table for the app's platform is refused.

Before the script runs, the target fills these placeholders — and the same values are in the script's process as `QA_FLOW_RELEASE_<FIELD>` / `QA_FLOW_RUNNER_<FIELD>` (`QA_FLOW_RELEASE_APP`, `QA_FLOW_RUNNER_HUB_URL`, …):

| Placeholder | Value |
|---|---|
| `{{release.app}}` | the build to install: on `browserstack` / `lambdatest` the provider's id (`bs://…` / `lt://…`) — **uploaded on the step's first run** and cached (by file checksum, re-uploaded after 29 days); locally the build's file path; for refs, the ref for the target's provider |
| `{{release.app_id}}` | the bundle id / package (the release's override, else the build's) |
| `{{release.url}}` | the website (web) |
| `{{release.platform}}`, `{{release.name}}`, `{{release.key}}` | the step's app's platform; the run's release name and key |
| `QA_FLOW_RUNNER_HUB_URL` | the WebDriver hub **with credentials** `open_driver()` connects to: BrowserStack `hub-cloud.browserstack.com/wd/hub`, LambdaTest `hub.lambdatest.com` (web) / `mobile-hub.lambdatest.com` (apps); locally the WebDriver server of the script's browser for a website (`LOCAL_WEBDRIVER_URL_<BROWSER>`, else — Chrome — `LOCAL_WEBDRIVER_URL`, else the driver's default: ChromeDriver `http://127.0.0.1:9515`, msedgedriver `:9516`, geckodriver `:4444`, safaridriver `:4445`) and the Appium server for an app (`APPIUM_URL`, else `http://127.0.0.1:4723`). Env only: `{{runner.hub_url}}` in a script is refused (*Opening the session*) |
| `QA_FLOW_RUNNER_CDP_URL` | the Playwright endpoint `open_page()` connects to with the URL-encoded caps (`wss://cdp.browserstack.com/playwright?caps=` / `wss://cdp.lambdatest.com/playwright?capabilities=`); empty locally (the browser is launched here). Env only: `{{runner.cdp_url}}` in a script is refused |
| `{{runner.username}}`, `{{runner.access_key}}`, `{{runner.name}}` | the provider's credentials (from `BROWSERSTACK_USERNAME` / `BROWSERSTACK_ACCESS_KEY` or `LT_USERNAME` / `LT_ACCESS_KEY`) and its name |

Credentials stay the environment's variables — a missing one fails the step with its name before anything starts, as does a build upload the provider refuses, or a run release with no build for the step's app. The step result records the release it ran (scheme, key, name, app, platform, provider it ran on, target, build and its checksum).

### The three must agree

Saving (and `validate_flow`) refuses a ui step unless:

| Check | Refused when |
|---|---|
| Engine + language | the engine is unknown, or isn't scripted in that language |
| Language + script | the script doesn't parse in the language (Python syntax; flow placeholders count as values) |
| Engine + script | the script doesn't import the engine's client — `appium` for Appium, `playwright` for Playwright. A script that imports only the *other* engine's client is named as such |
| Session | the script opens a session itself — `{{runner.hub_url}}` / `{{runner.cdp_url}}` (or `QA_FLOW_RUNNER_HUB_URL` / `_CDP_URL` read directly), `webdriver.Remote` (incl. a subclass that joins a session), `webdriver.Chrome(` and the other local drivers, a Playwright `chromium` / `firefox` / `webkit` `.launch(` / `.launch_persistent_context(` / `.connect(` / `.connect_over_cdp(`: *script opens its own session (<marker>) — open the session with open_page() / open_driver() (qa_flow_session) — the run's target decides where it connects* (dashboard, MCP and preflight) |
| Lookups | a lookup not every driver can run — an iOS predicate / class chain comparing `placeholderValue` (`placeholderValue == …`, `BEGINSWITH`, `CONTAINS`, …; comment lines aside): *script finds an element by a lookup not every driver has (placeholderValue) — placeholderValue isn't available on every iOS driver (TestMu's WebDriverAgent has none) — match the empty field's `value`, or better an accessibility id* (dashboard, MCP and preflight; see *Portable lookups*) |
| Target | `scheme` / `app` is missing or unknown; the app isn't one of the scheme's; the scheme has no targets; the flow's target (pin, else `default_target`) doesn't bind the app; the step still names `release` / `platform`; the engine isn't in the support table for the app's platform or can't run on the target's provider (*What may run where*); a `{{release.*}}` / `{{runner.*}}` names a field that doesn't exist. The release is checked when the flow runs, not on save |
| Engine + options | the engine is `maestro` (removed); `runs_on`, `executor`, `device` or `cloud` is on the step (the flow's target supplies them); `options` is not an object, names an option the catalog doesn't list, or sets one outside its engine / language / platform / provider scope or of the wrong kind |

The import has to be in the script itself (`from appium import webdriver`, `from playwright.sync_api import …`): a script that reaches the client only through its own helper module is refused. Nothing is checked about what is installed — that is the run machine's business (see Prerequisites).

| Engine | Languages | What makes a script the engine's |
|---|---|---|
| `appium` | `python` | imports `appium` (Appium-Python-Client) |
| `playwright` | `python` | imports `playwright` |

**The script is the user's.** Save it as given. Do not rewrite it into another form, and do not invent locators, capabilities or waits it doesn't contain. The only edits to propose are the ones this page calls for: `{{env.*}}` in place of hardcoded per-environment or secret values, `{{context.*}}` for values from earlier steps, and the exports file for values later steps need.

## What the script gets

- **Placeholders, in the text.** The target's `{{release.*}}` / `{{runner.*}}` (*The target*), `{{env.NAME}}`, `{{context.<step>.<field>}}` (incl. `{{context.flow_input.*}}`) and `{{auto.*}}` are replaced in the script text before it runs: `EMAIL = "{{env.LOGIN_EMAIL}}"`. One that doesn't resolve fails the step before anything starts. Other `{{ }}` in the script (an f-string's `{{literal}}`) are left alone. The value is pasted as text — put it inside the script's own quotes.
- **`QA_FLOW_EXPORTS`** — path of a file the script may write a JSON object to (see below).
- **`QA_FLOW_CAPS`** — a JSON object of the provider's capabilities for this step, built by QA Flow from the target's provider profile (`Common/ui/providers.py`: one record per provider holding its vendor options key, the keys allowed in it, its defaults and its browsers): network logging on, real devices, the permission alerts accepted the provider's own way (iOS `autoAcceptAlerts`, Android `autoGrantPermissions`: `appium:` on Local and BrowserStack, inside `lt:options` on LambdaTest), build `qa-flow`, session name = the flow's name, the device (the target's, else `{{env.BROWSERSTACK_<PLATFORM>_DEVICE}}` / `_VERSION`, `{{env.LT_<PLATFORM>_DEVICE}}` / `_VERSION`, `{{env.<PLATFORM>_UDID}}`), the binding's own `caps` merged over them. **A script never names a provider**: it keeps only W3C and `appium:` capabilities (`open_driver(caps={...})`) and the helper merges the provider's over them (`open_page()` sends them to the endpoint). **The browser is the helper's argument** — `open_page("webkit")`, `open_driver("firefox")` (`chrome`, `firefox`, `edge`, `safari`; Playwright `chromium` / `webkit` too; none = `chrome`), else the script's first literal `"browserName": "<x>"`: the run reads it and the provider's profile turns it into its own name, OS and options (Safari → `MacOS Tahoe` on LambdaTest, `OS X` on BrowserStack; Playwright WebKit → `pw-webkit` / `playwright-webkit`). Every web WebDriver session is asked for WebDriver BiDi (`"webSocketUrl": True`; on BrowserStack with `seleniumBidi`) when its browser speaks it — all but Safari — for the response capture (*API responses*). The run also reads `se:cdp` / `devtools_address(` (the session's DevTools address), and a script that opens a WebDriver session (`open_driver(`) gets WebDriver caps whatever its engine. Credentials come as `{{runner.username}}` / `{{runner.access_key}}`. **A script never calls the provider's own API** (`{{runner.api_url}}`, `browserstack_executor`, `lambdatest_executor`): refused on save like a provider marker (*script calls the provider's own API (…)*) — QA Flow reads network logs itself. **Validated per provider, three times:** a script containing another provider's marker — a vendor options key (`lt:options` / `LT:Options` / `bstack:options`), a `browserstack.*` key or host, a `lambdatest.com` host, an `LT_*` / `BROWSERSTACK_*` env var — is refused on save (dashboard and MCP: *script names provider '<p>' (<marker>) — capabilities come from the flow's target via QA_FLOW_CAPS*); a binding's `caps` is checked against its provider's profile when the scheme is saved; and the caps built for a run are checked again before the script starts (a foreign or disallowed key, or a browser the provider doesn't offer, fails the step). A missing env var the caps use fails the step with its name.
- **`QA_FLOW_RESPONSES`** / **`QA_FLOW_RESPONSE_URLS`** — where the capture helper writes the declared calls, and the URL parts the step declared (see *API responses*).
- **`qa_flow_session.py`** — the session helpers, next to the script (also in a HyperExecute job): `from qa_flow_session import open_page, open_driver` (*Opening the session*).
- **`qa_flow_capture.py`** — the capture helper, next to the script (also in a HyperExecute job): `from qa_flow_capture import capture`.
- **`QA_FLOW_ARTIFACTS_DIR`** — a directory for this run; save screenshots there.
- **`QA_FLOW_STEP_NAME`** — the step's name.
- **`QA_FLOW_RELEASE_*` / `QA_FLOW_RUNNER_*`** — the target's values (*The target*).
- The run's environment variables, as every other step sees them.

The app, its bundle id / package and the site URL come from the release — `{{release.*}}`, never literals; the hub / endpoint are the helper's. Other values that differ per environment (device UDID or cloud device name, test account) belong in env vars — `{{env.*}}`; see `ai-rules/negotiation/env-vars.md`.

## What the script gives back

- **Exit code.** `0` passes the step; anything else fails it, and the last lines of the script's output become the step's error. A Python script fails by raising (`assert …`, any exception).
- **Exports.** To hand values to later steps, write a JSON object to the exports file:

  ```python
  import json, os
  json.dump({"balance": balance_text}, open(os.environ["QA_FLOW_EXPORTS"], "w"))
  ```

  Each key becomes `{{context.<step name>.<key>}}` (and `{{context.<key>}}`). Exports of a failed step are dropped. A script may instead print one line per export — the marker `QA_FLOW_EXPORT`, then a JSON object — to its output; several lines add up, a later line wins on a shared name, and the exports file wins over them.
- **Log.** stdout and stderr are saved as `<step name>.log` under `Reports/ui/<flow>/<timestamp>/` (`REPORTS_DIR` / `QA_FLOW_UI_ARTIFACTS_DIR`); the step error names the file. Screenshots are the script's to take — nothing is captured automatically.

## API responses

A step can take values from the API calls its app or page made, and show those calls in the run results. The step **declares** the calls; the script **hands them over**.

```json
"responses": [
  {"name": "verify_otp", "url": "/api/auth/verify", "method": "POST",
   "export": {"access_token": "body.data.token",
              "request_id": "headers.x-request-id",
              "verify_status": "status"}}
]
```

- `name` — how the call is labelled in the results; unique in the step. `url` — a part of the call's URL (plain text, not a pattern; `{{env.*}}` / `{{context.*}}` allowed). `method` — optional; any method matches without it.
- `export` — optional `{variable: source}`. A source is `status`, `headers.<name>` (any letter case), `body` (all of it) or `body.<path>` — dots, a number indexes a list (`body.items.0.id`). Each variable becomes `{{context.<step name>.<variable>}}` (and `{{context.<variable>}}`), like the script's own exports; on a shared name the declared export wins.
- When several calls match, the **last** one is used.
- A declared call that never happened, or a source that is not in it, **fails the step** — the error lists the calls that were seen — and, as for any failed step, nothing is exported. An entry without `export` is only shown, but must still happen.
- Only the declared calls are kept. They are saved as `<step name>.responses.json` beside the step's log, **bodies included** — a token in a response is on disk in plain text there.

**Handing the calls over — one line in the script, the same locally and on every provider.** QA Flow cannot see a session it did not open, so the script hands the calls over:

- **Web (Appium / Selenium WebDriver, or Playwright):** wrap the driving in the capture helper QA Flow puts next to the script. It keeps the responses whose URL contains a declared `url` (`QA_FLOW_RESPONSE_URLS`), bodies JSON-parsed, and writes them to `QA_FLOW_RESPONSES` when the block ends — after waiting up to 20 s for a declared call still on its way.

  ```python
  from qa_flow_capture import capture

  with capture(driver):      # WebDriver: WebDriver BiDi network events, bodies through network.getData
      ...                    # drive the site
  with capture(page):        # Playwright: page.on("response")
      ...
  ```

  A WebDriver session needs WebDriver BiDi: the web targets ask for it (`"webSocketUrl": True`) on Chrome, Edge and Firefox. A session the script **joined** (`open_driver(continue_session=…)`) has no capabilities: the step that opened it exports its BiDi address (`open_driver(keep_open=True)` exports `bidi_url`), and the joining step passes it — `capture(driver, bidi_url="{{context.request_otp.bidi_url}}")`. **Safari has no WebDriver BiDi**: drive it with Playwright WebKit (`"browserName": "webkit"`); a WebDriver session without BiDi makes `capture` fail with that message. With Playwright, a body is read while Playwright has control — wait for the page to react (a locator wait) before navigating away. No DevTools (CDP) route.
- **Native app (Appium):** nothing to write — `open_driver()` prints the session id (`QA_FLOW_SESSION <id>`, also for a joined session); QA Flow reads that session's network log on the target's provider once the script has ended (polling up to 5 minutes — the providers write it after the session ends) and matches the declared calls in it.

  LambdaTest (real devices) and BrowserStack keep the response bodies in their mobile logs; a local device has no log. BrowserStack's **web** log has bodies too (`networkLogsOptions.captureContent`, set by its profile), so a BrowserStack Safari WebDriver step may hand over its session id instead of capturing; LambdaTest's web log has none (checked 2026-10-05) — the step fails with a message pointing to `capture`.

Calls the script writes to `QA_FLOW_RESPONSES` itself win over the network log.

## Opening the session

QA Flow puts `qa_flow_session.py` next to the script; the script opens its page or driver with it and never itself — **one script per step, never one per provider**:

```python
from qa_flow_session import open_page, open_driver, devtools_address

with open_page("webkit") as page:          # Playwright: the provider's endpoint, else a local browser; closed at the end
    page.goto("{{release.url}}")

with open_driver(caps={"platformName": "Android", "appium:automationName": "UiAutomator2"}) as driver:
    ...                                    # WebDriver / Appium on the target's hub; quits at the end
driver = open_driver("firefox")            # or keep it in a variable and quit it yourself
```

- **`open_page(browser="chrome", driver=None, **launch)`** — a Playwright page. On a provider it connects to its endpoint with the target's caps; on Local it launches Playwright's own browser (chrome / chromium / edge → Chromium, firefox → Firefox, webkit / safari → WebKit; `**launch` go to `launch()`, e.g. `headless=False`); the browser is closed when the block ends. **Local needs nothing running.** `open_page(driver=driver)` drives a WebDriver session's Chromium with Playwright (its `se:cdp` / `debuggerAddress`); leaving the block only disconnects.
- **`open_driver(browser=None, caps=None, keep_open=False, continue_session=None, session_caps=None, accept_alerts=True)`** — an Appium WebDriver session on the target's hub with the script's `caps` and the target's over them (`browserName` set for a website; Edge as `MicrosoftEdge`). It prints `QA_FLOW_SESSION <id>` itself. **It accepts the system alerts of an app session** (see *System alerts*). **Where it connects locally:** the browser's WebDriver server for a website — ChromeDriver at `LOCAL_WEBDRIVER_URL` / `LOCAL_WEBDRIVER_URL_CHROME` (default `:9515`), msedgedriver at `LOCAL_WEBDRIVER_URL_EDGE` (`:9516`), geckodriver at `LOCAL_WEBDRIVER_URL_FIREFOX` (`:4444`), `safaridriver -p 4445` at `LOCAL_WEBDRIVER_URL_SAFARI` (`:4445`), so steps on different browsers run side by side — and Appium at `APPIUM_URL` for an app. **The built-in Local provider starts what doesn't answer itself** — the driver, Appium and the emulator / simulator (never a real device) — and reuses what does (see *Prerequisites*). **A copy of Local still needs the server running:** its preflight tries the address the step will use (at most 1 s) and refuses the run with *nothing is listening at <url> — start Appium / ChromeDriver, or pick a cloud target*. Cloud hubs are never probed.

### System alerts

**A script never handles a system permission alert** (notifications, location, camera, …) — `open_driver()` does, on every target:

- the target's caps ask the provider to accept them (iOS `autoAcceptAlerts`, Android `autoGrantPermissions`; not every provider honours that — TestMu doesn't always);
- a **new app session** is checked once right after it starts;
- **every element lookup** of an app session — new or joined (`continue_session`) — first checks for one, at most once a second, so an alert that shows later (after a splash, after logging in) is accepted while the script's wait loop runs. A website is never checked.

Only **permission** alerts are accepted: on iOS (`mobile: alert`) with the button *Allow While Using App*, *Allow* or *Allow Once* — *OK* only when the alert's text is a system permission prompt (*"… Would Like to …"*); any other alert (the app's own: an error *OK*, a rate-the-app prompt) is left alone, and so is a permission alert on a device in another language (its labels don't match — accept it in the script). On Android the permission controller's allow button, by id (`permission_allow_button` / `permission_allow_foreground_only_button`, `permissioncontroller` or the older `packageinstaller`), else by its *Allow* / *While using the app* text on that package only. An alert still showing after 5 accepts is left, with one log line. Nothing to accept is not a failure.

- To **wait** for one explicitly: `from qa_flow_session import accept_system_alerts` and `accept_system_alerts(driver, seconds=5)` — waits up to `seconds` for the first, ends when none is left; never fails.
- A test that **checks the alert itself**: `open_driver(..., accept_alerts=False)` — no checks at start or on lookups, and the target's auto-accept / auto-grant are switched off.
- App-level prompts (a rate-the-app "Not Now", an intro's "Skip", the app's own dialogs) are the script's, as before.

### Portable lookups

Find elements the way **every** driver can: an **accessibility id** first (`AppiumBy.ACCESSIBILITY_ID`), else an id / predicate on `name`, `label`, `type` or `value`. **Never `placeholderValue`** — TestMu's WebDriverAgent has no such attribute (a predicate naming it fails whole); an empty text field's `value` is its placeholder on every driver: `"type == 'XCUIElementTypeTextField' AND value == 'Email address'"`. A predicate / class chain naming `placeholderValue` is refused on save (*Lookups* in the checks table).

### Continuing where another step or flow stopped

- **WebDriver / Appium session across steps:** the first step opens it with `open_driver(..., keep_open=True)` (in a `with` block it is then quit only when the block fails; the helper exports `session_id` and, with BiDi, `bidi_url`); a later step on the same target joins it — `open_driver(continue_session="{{context.request_otp.session_id}}")`, naming the same browser as the opener (`open_driver("safari", continue_session=…)`: locally each browser has its own driver) — and quits it at the end. Works on every target (on Local the WebDriver / Appium server keeps the session between the steps).
- **Playwright: a page lives in one step, on every target.** A provider's endpoint browser can't be joined by a later connection, and the same approach is used locally — `open_page(keep_open=…)` / `open_page(continue_session=…)` are refused at save and at run (*a Playwright page can't be kept open or continued by a later step … put the steps in one Playwright step, or use open_driver() (WebDriver / BiDi), which continues a session on every target*). Read a value a query step would give inside the script (a saved query, as the OTP flows do), or drive the browser over WebDriver: `open_driver("chrome", keep_open=True)` + `open_page(driver=driver)` in the first step, `open_driver(continue_session=…, session_caps={"se:cdp": "{{context.<step>.cdp_url}}"})` + `open_page(driver=driver)` in the next.
- **Mobile app across sessions:** open the session without resetting or relaunching the app — `appium:noReset: true` and `appium:autoLaunch: false` (iOS also `appium:shouldTerminateApp: false`; Android `appium:dontStopAppOnReset: true`). The app is then on the screen the previous script left it on — in the same flow or in a later one.

A script that should start clean does the opposite on purpose (terminate + activate the app, a fresh browser).

Each mobile script pays the driver's session start. Put actions that belong together in **one** script rather than one step per tap; split a script only where another step type (a query, an API call) has to run in between.

## Appium as a HyperExecute job (a target binding's `executor: "hyperexecute"`)

An Appium step whose target runs its app on `lambdatest` normally runs its Python script on this machine, which talks to LambdaTest's hub. When that binding also sets `executor: "hyperexecute"` (a target of the scheme — e.g. *Android on LambdaTest via HyperExecute job*; the step itself names no executor) the same script — placeholders already filled in — runs as a HyperExecute job on a TestMu Windows VM (`Common/ui/hyperexecute_appium.py`):

- **What comes back is the same.** A wrapper on the VM gives the script its own `QA_FLOW_EXPORTS` / `QA_FLOW_RESPONSES` / `QA_FLOW_ARTIFACTS_DIR` (and `qa_flow_capture.py` / `qa_flow_session.py` beside it) and prints what it wrote; QA Flow turns that into the step's exports, its `responses` calls (or the network log of the `QA_FLOW_SESSION` it printed) and its screenshots. The script also gets the step's `QA_FLOW_STEP_NAME` / `QA_FLOW_RELEASE_*` / `QA_FLOW_RUNNER_*`. The step passes or fails on the script's own exit code; the job link and the script's output are in the step log.
- **The VM is not this machine.** It can't reach a VPN database or the dashboard, so a value only this machine can read (an OTP from MongoDB) must come from a **query step between two ui steps** — the flow runs that step here, between the jobs. A one-step script that reads the database mid-run does not work as a job.
- **Sessions carry over by id.** Each step is its own job on a new VM; a session the first script leaves open (`open_driver(keep_open=True)`) is joined by the next (`open_driver(continue_session="{{context.<step>.session_id}}")`), as from this machine. The time between the two jobs (VM setup, about 1 to 3 minutes, plus the steps in between) counts as **idle time on the device**: a long gap ends the session with `idle_timeout`.
- **What the VM has.** Python of the same minor version as the flow's, and the `Appium-Python-Client` / `selenium` / `requests` versions installed here; a script importing anything else fails on the VM. The script's resolved text — including any `{{env.*}}` secret it uses — and `QA_FLOW_RUNNER_HUB_URL` with its credentials travel to TestMu with the job.
- **Prerequisites:** `LT_USERNAME` / `LT_ACCESS_KEY` and the HyperExecute CLI — `hyperexecute` on the PATH, `HYPEREXECUTE_BIN`, or downloaded once into `~/.hyperexecute/`; the plan must include HyperExecute and real devices. Jobs run on Windows VMs: LambdaTest's hub refused every new device session from a Linux VM (2026-10-22).

When negotiating, it is a choice of the flow's target (`negotiation/flow.md` question 4), never a step setting: offer such a target with its cost — slower steps (VM setup per job) and no access to this machine's network. A binding's executor applies only to the steps it can run (Appium + Python); the others on that target use the provider directly.

## Failures and results

- The flow stops at the failing step (no continue-on-failure).
- `failed` = the script ran and exited non-zero; `error` = it never got to decide: the language's interpreter couldn't start, the script timed out, or the exports file isn't a JSON object.
- A step saved in an old form (`actions` / `target`, or `runner` without `engine` + `language`) is refused when the flow is saved or validated.

## Prerequisites (tell the user — QA Flow starts servers and devices, but never installs these)

The script's own requirements, on the machine that runs the flow:

- **Python + Appium:** the `mobile` extra (`Appium-Python-Client`), Appium 3 on the PATH with the XCUITest (iOS) or UiAutomator2 (Android) driver, the Android SDK with `ANDROID_HOME` set / Xcode (`xcrun`), and the AVD / simulator (or a connected real device). iOS on Local, when XCUITest can't build WebDriverAgent with the installed Xcode: set `IOS_WDA_DERIVED_DATA` to a prebuilt WDA's derived data (`xcodebuild build-for-testing … -derivedDataPath <dir>`) and Local iOS sessions get `appium:usePrebuiltWDA` + `appium:derivedDataPath` (`docs/device-providers.md` → *Local runs*).
- **Python + Playwright:** the `web` extra (`playwright`) and `playwright install chromium`.
- **WebDriver websites:** the browser and the `selenium` package (Selenium Manager fetches its driver); Safari needs `safaridriver --enable` run once.
- On the built-in **Local** target a run starts what is missing and reuses what answers — the WebDriver server, Appium, the emulator / simulator (never a real device) — and installs the release's build per the binding's `install`; it never installs a toolchain. What it started keeps running for the next run; `qa-flow local status` / `qa-flow local stop` (or *Stop local devices* on the Device Providers page's Local card) show and stop it — only what QA Flow started. Details: `docs/device-providers.md` → *Local runs*.
