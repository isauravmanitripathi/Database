```python
# file path: src/main.py
import os
import logging
from flask import Flask
from config.Config import getBrokerAppConfig, getServerConfig, getSystemConfig
from restapis.HomeAPI import HomeAPI
from restapis.BrokerLoginAPI import BrokerLoginAPI
from restapis.StartAlgoAPI import StartAlgoAPI
from restapis.PositionsAPI import PositionsAPI
from restapis.HoldingsAPI import HoldingsAPI
```

For bootstrapping the trading runtime, the file brings in os so the launcher can read environment variables and perform basic path or process operations at startup, and logging so the entrypoint can configure and emit the runtime diagnostics and audit trail needed during strategy execution. It imports Flask because the entrypoint creates the web application that exposes the control and status REST surface used to interact with the engine. The three configuration accessors getBrokerAppConfig, getServerConfig and getSystemConfig are pulled from Config so the entrypoint can load broker-specific credentials and mappings, server binding and runtime HTTP behavior, and global system flags before initializing services and the strategy. The REST handler classes HomeAPI, BrokerLoginAPI, StartAlgoAPI, PositionsAPI and HoldingsAPI are imported so the Flask app can wire up the control endpoints that let an operator land on the home page, perform broker login, kick off BaseStrategy.run, and query live positions and holdings. This follows the project pattern of modules importing logging and configuration helpers; compared with other modules that import flask.views.MethodView and core.Controller to implement view logic or modules that pair getBrokerAppConfig with broker-specific login classes, the entrypoint focuses on creating the Flask app and wiring in prebuilt API handlers while also loading server and system configuration because it coordinates both the HTTP control surface and the runtime environment.

```python
# file path: src/main.py
app = Flask(__name__)
```

This line creates the Flask application object named app using the Flask class so the entrypoint can expose an HTTP surface for runtime monitoring and control; in the context of the entrypoint that bootstraps server and broker configuration and then calls BaseStrategy.run, app becomes the WSGI app and request dispatcher that will host health, metrics, and control endpoints the trading runtime relies on. Later lines interact with this same app to set runtime options via app.config, to wire endpoints like HomeAPI with add_url_rule, and to start the server with app.run, so this instantiation is the foundational piece that makes those web-facing integrations possible while the strategy and execution components are running.

```python
# file path: src/main.py
app.config['DEBUG'] = True
```

After the Flask application object is created earlier, the code sets the Flask application's DEBUG configuration flag to True, which tells the Flask runtime to run in debug mode for this process. In practical terms this enables the development-time conveniences that Flask exposes: more verbose exception tracebacks and the automatic reloader so changes to application code restart the server without a manual restart. Within the entrypoint's bootstrapping sequence this is one of several environment-configuration steps—alongside initializing logging with initLoggingConfg and emitting the serverConfig via logging.info—that prepare the runtime so the rest of the startup (loading broker and server configurations and eventually invoking BaseStrategy.run) runs with development-friendly behavior.

```python
# file path: src/main.py
app.add_url_rule("/", view_func=HomeAPI.as_view("home_api"))
```

When the entrypoint finishes setting up the runtime it registers an HTTP route on the Flask application by wiring the base URL to the HomeAPI class-based view so that requests to the server root are handled by HomeAPI; the view is exposed under the logical name home_api so other parts of the system or external tools can reference it. This is the same registration pattern used elsewhere in the file—add_url_rule binds a path to a class-based view produced via as_view—however this particular registration targets the base '/' endpoint (typically used for a landing page or health/status check), whereas the similar registrations for HoldingsAPI, PositionsAPI and StartAlgoAPI expose specific runtime and control endpoints for holdings, positions and starting the algorithm respectively.

```python
# file path: src/main.py
app.add_url_rule("/apis/broker/login/zerodha", view_func=BrokerLoginAPI.as_view("broker_login_api"))
```

During application bootstrap the entrypoint wires up an HTTP route that exposes the Zerodha broker login flow: it registers the /apis/broker/login/zerodha path so incoming requests for broker authentication are dispatched to the BrokerLoginAPI class-based view (registered under the logical name broker_login_api). This follows the same routing pattern used for other endpoints like HoldingsAPI, PositionsAPI, and HomeAPI, where add_url_rule maps paths to view classes created via as_view; the difference here is that BrokerLoginAPI is responsible for initiating and handling the Zerodha-specific authentication sequence so the broker adaptor can obtain or refresh credentials that the connectivity and execution layers need before live trading begins. The route is set up during the same server bootstrapping phase when logging, configuration, and other API endpoints are initialized, ensuring the UI or automated components can trigger broker login as part of startup or runtime operation.

```python
# file path: src/main.py
app.add_url_rule("/apis/algo/start", view_func=StartAlgoAPI.as_view("start_algo_api"))
```

As the entrypoint exposes HTTP controls to the runtime, this line registers a route that lets external clients trigger the algo engine to start: it maps the URL for starting an algorithm to the StartAlgoAPI view handler and gives that handler the logical name start_algo_api. In the flow of the application this endpoint is the operator-facing control that transitions the system from initialized to active by invoking the strategy execution path (eventually calling BaseStrategy.run) so the strategy can subscribe to market data, evaluate signals, and submit orders through the broker adapter. This follows the same registration pattern used for the other endpoints in the file (HomeAPI, PositionsAPI, HoldingsAPI), but unlike those read-oriented endpoints, the StartAlgoAPI performs a state-changing orchestration to kick off live trading.

```python
# file path: src/main.py
app.add_url_rule("/positions", view_func=PositionsAPI.as_view("positions_api"))
```

The line registers an HTTP route on the Flask application that exposes the runtime's current trade positions via the PositionsAPI class-based view. During startup this wiring converts PositionsAPI into a callable endpoint by using its as_view factory and binds that handler into the app's routing table so external consumers—like the web dashboard, monitoring tools, or internal REST clients—can request the live positions while the strategy is running. This follows the same registration pattern used for HoldingsAPI and HomeAPI (and for BrokerLoginAPI on the broker login route): each call turns a class-based view into a handler and maps a specific path and endpoint name into the server during bootstrap; the PositionsAPI registration specifically provides access to the active positions state rather than holdings, the home page, or broker login flows.

```python
# file path: src/main.py
app.add_url_rule("/holdings", view_func=HoldingsAPI.as_view("holdings_api"))
```

During startup the entrypoint registers an HTTP route that exposes current portfolio holdings to external callers by wiring the /holdings path to the HoldingsAPI class-based view. Concretely, the app creates a callable from HoldingsAPI and attaches it under an endpoint name so the server can route incoming requests to that view; the view in turn will fetch and serialize the live holdings data from the trade lifecycle layer or broker adaptor so dashboards and monitoring tools can query the running system. This follows the same registration pattern used for the positions endpoint (PositionsAPI), the root/home endpoint (HomeAPI), and the broker login endpoint (BrokerLoginAPI) — the difference is purely the route and the domain responsibility: holdings returns aggregated owned instruments and cash balances, positions returns active trade positions, home serves the base UI/health, and broker login handles authentication. The route is set up during bootstrap so the API surfaces are available by the time the strategy is started (BaseStrategy.run), enabling real-time observability and control while the trading runtime executes.

```python
# file path: src/main.py
serverConfig = getServerConfig()
```

The entrypoint stores the runtime configuration into the serverConfig variable by calling getServerConfig so the process has a single, concrete set of server settings to drive initialization. getServerConfig performs file I/O: it reads the server.json file from the project’s config area and returns the parsed JSON as a dictionary, and assigning that return to serverConfig makes those values available to the rest of the startup sequence. Those values are then used immediately to configure runtime details such as the deployment directory, logging and server port (you’ll see later where deployDir and port are read and where serverConfig is logged), and they are the authoritative source for helper functions that persist or load runtime state—getTimestampsData and saveTimestampsData use serverConfig to build the timestamps file path, and other subsystems like Instruments.fetchInstrumentsFromServer and TradeManager.registerStrategy also call getServerConfig when they need the same environment paths. In short, the assignment captures the server configuration early so the entrypoint and subsequent components, including the call into BaseStrategy.run, operate with a consistent configuration context.

```python
# file path: src/config/Config.py
import json
import os
```

The configuration loader brings in json to parse and emit the JSON blobs that represent server settings, broker app settings and the persisted timestamps used across authentication, controller and instruments logic, and it brings in os to interact with the host environment and filesystem — locating config files, expanding paths, creating or checking directories, and reading/writing the timestamp files. Other modules in the codebase follow a similar pattern but add more dependencies: for example, controller or startup code typically imports os and json alongside logging and then pulls in functions from config.Config such as getServerConfig, getTimestampsData and saveTimestampsData to access the data this file provides; orchestrator code imports Controller and Utils to wire components, and model-focused modules import types like Segment and ProductType. The minimal import set here reflects that Config’s responsibility is low-level file and environment handling for the larger system.

```python
# file path: src/config/Config.py
def getServerConfig():
  with open('../config/server.json', 'r') as server:
    jsonServerData = json.load(server)
    return jsonServerData
```

getServerConfig loads and parses the server configuration JSON stored in the project's config directory and returns that configuration as a Python dictionary. It provides the concrete server settings the process needs at startup—serverConfig is populated by calling getServerConfig and then used throughout the runtime—so values like the deployment directory are made available to other utilities. getTimestampsData and saveTimestampsData both call getServerConfig to derive the path where timestamps.json is read and written, and many other components (for example Instruments.fetchInstrumentsFromServer, TradeManager.registerStrategy, ticker and login-related code) call getServerConfig when they need runtime paths or flags. The function performs simple file I/O and JSON parsing and follows the same straightforward pattern used by getSystemConfig and getBrokerAppConfig for loading their respective JSON blobs.

```python
# file path: src/main.py
deployDir = serverConfig['deployDir']
```

With serverConfig already populated by getServerConfig, the entrypoint pulls out the deployDir value and stores it in the local deployDir variable so the rest of the startup sequence has a canonical filesystem location to work with. That extracted deploy directory is used by subsequent initialization steps to locate or create runtime artifacts (deployment-specific config, broker adapters, timestamp files, and any data folders the engine expects) and to resolve paths when the process sets up its environment and logging. This is the same pattern used a few lines later for logFileDir and is immediately echoed for visibility by the run-time print of the deployDir so operators can confirm which deployment path the runtime is using.

```python
# file path: src/main.py
logFileDir = serverConfig['logFileDir']
```

After getServerConfig populated serverConfig with the runtime settings, this line reads the configured log file directory from serverConfig and assigns it to the local variable logFileDir so the entrypoint can use that path when initializing logging and any filesystem steps that follow (like creating the directory or pointing log handlers). This follows the same pattern used for other environment values such as deployDir and port, and the value is later echoed to the console by the print statement you already saw.

```python
# file path: src/main.py
print("Deploy  Directory = " + deployDir)
```

After serverConfig was loaded into serverConfig and deployDir was extracted from it earlier, this line prints a labeled, human-readable echo of the deployDir value to standard output so an operator or startup log can immediately see which deployment directory the runtime will use. In the startup narrative of the entrypoint that bootstraps configuration, environment and logging before handing control to BaseStrategy.run, this print serves as an early, side-effect-free confirmation of a critical path setting — the same simple read-then-echo pattern is used for logFileDir a few lines later (logFileDir is read from serverConfig and then printed), but these prints write to stdout as immediate startup feedback prior to the full logging subsystem being initialized.

```python
# file path: src/main.py
print("LogFile Directory = " + logFileDir)
```

After serverConfig was loaded and logFileDir was pulled out of it, the entrypoint emits the logFileDir value to the process console so an operator or log collector can immediately see where runtime logs will be written; this is the same pattern used when the code prints deployDir earlier, and it happens just before initLoggingConfg is invoked with a logfile path composed from that directory, making the printed line a simple startup visibility checkpoint that ties the loaded configuration to the logging initialization that follows.

```python
# file path: src/main.py
initLoggingConfg(logFileDir + "/app.log")
```

After serverConfig is read and the logFileDir value is printed, the call to initLoggingConfg with the path built from logFileDir and the app.log filename establishes the process-wide logging sink that the rest of the runtime will use. initLoggingConfg configures Python’s root logging so that messages are written to that file, formats each entry to include a timestamp followed by the textual message, and sets the severity threshold to INFO with a human-readable date format. The function itself returns nothing; its purpose is to ensure that subsequent logging calls such as the logging.info call that follows will end up persisted in the configured log file under the runtime’s log directory. In the context of the algo-trade framework, this guarantees that broker adapters, market data ingestion, strategy execution and the order manager produce a consolidated, timestamped audit trail for debugging, monitoring and post-trade analysis.

```python
# file path: src/main.py
def initLoggingConfg(filepath):
  format = "%(asctime)s: %(message)s"
  logging.basicConfig(filename=filepath, format=format, level=logging.INFO, datefmt="%Y-%m-%d %H:%M:%S")
```

initLoggingConfg is a small bootstrapping helper that centralizes how the process emits runtime logs: when the entrypoint hands it the path to the application log file it configures Python's logging system to write messages to that file, to include a standardized timestamp and the message body, and to filter output at the INFO level. Its effect is to make all subsequent logging calls—such as the earlier-recorded serverConfig and brokerAppConfig messages—land in a single, timestamped logfile with a consistent date format, so runtime events, authentication steps, strategy lifecycle messages and order activity are persisted for troubleshooting and auditing. Conceptually it is a thin wrapper around the standard logging configuration API that sets filename, message format, date format and log level so the rest of the framework can simply use logging.info/debug/error and rely on a consistent, file-backed logging destination.

```python
# file path: src/main.py
logging.info('serverConfig => %s', serverConfig)
```

As the application entrypoint initializes the runtime, the call to logging.info emits the contents of serverConfig into the process logs at INFO level so operators and developers can see exactly which server settings were loaded by getServerConfig. This gives immediate startup visibility — confirming environment flags, directory paths, log and persistence settings, and any other server-level options before those values are used to drive behavior. Because the route registrations you already saw depend on the runtime configuration and the port value is later read from serverConfig, having a recorded snapshot of serverConfig makes it straightforward to correlate what the service intended to use versus what it actually bound to. The line plays the same role as the similar logging of brokerAppConfig: both produce human-readable checkpoints of configuration state, while the later assignment of port consumes one of the values from the already-logged serverConfig.

```python
# file path: src/main.py
brokerAppConfig = getBrokerAppConfig()
```

After the earlier step that loaded serverConfig, the entrypoint calls getBrokerAppConfig to populate brokerAppConfig with the broker-specific application settings the runtime needs to initialize connectivity and authentication with an exchange adaptor. getBrokerAppConfig performs file I/O to read and parse the brokerapp.json configuration file and returns the resulting JSON object; the returned dictionary is then stored in brokerAppConfig, logged immediately afterward, and later consumed by the Controller helpers (including getBrokerName and the broker login handlers) as well as the broker adaptor initialization code so the strategy and execution layers start with the correct broker credentials, endpoints, and app-level flags. This follows the same pattern used earlier when the code loaded serverConfig: centralized JSON config is read at startup and kept in a single variable for the rest of the runtime to reference.

```python
# file path: src/config/Config.py
def getBrokerAppConfig():
  with open('../config/brokerapp.json', 'r') as brokerapp:
    jsonUserData = json.load(brokerapp)
    return jsonUserData
```

getBrokerAppConfig is the small startup helper that reads the broker application settings from the project's broker configuration file and returns them as a parsed JSON object for the rest of the runtime to consume. During initialization the entrypoint calls getBrokerAppConfig and stores its result into brokerAppConfig so the Controller and authentication flow have a single source of truth for broker name, client ID, app key, app secret and redirect information; BrokerAppDetails uses those values to populate its instance fields and the broker-specific login implementation such as ZerodhaLogin consumes them to perform the actual login handshake. The function performs simple file I/O: it opens the broker config file, parses the JSON, and hands the resulting dictionary back to the caller, following the same pattern used by getServerConfig for server settings.

```python
# file path: src/main.py
logging.info('brokerAppConfig => %s', brokerAppConfig)
```

At this point in the entrypoint, after initLoggingConfg has established the process-wide logging sink and after getBrokerAppConfig populated brokerAppConfig with the broker-specific settings, the logging.info call writes the contents of brokerAppConfig into the application log at INFO level so operators and developers can see exactly which broker application settings the runtime will use for connectivity and authentication; this mirrors the earlier serverConfig emission and provides the same startup traceability for broker-side configuration before the strategy is started.

```python
# file path: src/main.py
port = serverConfig['port']
```

The entrypoint extracts the port setting from serverConfig and stores it in the local variable named port so the startup sequence knows which TCP port the runtime should use when it brings up its network-facing components (for example the management/API endpoint or any telemetry socket the engine binds). This is the same configuration-read pattern you saw earlier where getServerConfig produced serverConfig and the logFileDir value was pulled out; here the code mirrors that approach by pulling another runtime parameter from the parsed server configuration so subsequent initialization steps can pass a concrete port value into the server/engine that will be started.

```python
# file path: src/main.py
app.run('localhost', port)
```

The app.run invocation starts Flask’s built-in HTTP server bound to the loopback interface using the port value that was read into port from serverConfig. Because app was instantiated earlier with Flask and its debug flag set, calling app.run hands control to Flask’s request loop so any routes and view functions registered on the Flask application become live and begin handling incoming HTTP requests. In the context of the startup sequence that already initialized logging with initLoggingConfg and loaded broker settings with getBrokerAppConfig, app.run makes the runtime’s HTTP control and observability surface reachable locally so operators, the dashboard, health checks, or other local tooling can query runtime state, positions, and control endpoints while the strategy and execution components are running. The call therefore both opens the local network endpoint for the service and keeps the process running so BaseStrategy.run and the rest of the trading runtime remain active and reachable.

```python
# file path: src/main.py
if os.path.exists(deployDir) == False:
  print("Deploy Directory " + deployDir + " does not exist. Exiting the app.")
  exit(-1)

```

After deployDir is pulled out of serverConfig, the entrypoint verifies that the path actually exists on disk; if it does not, the process prints a human-readable message naming the missing deployDir and then terminates with a non-zero exit status so the startup stops immediately rather than proceeding into a partially-initialized runtime. This is an early startup validation that protects later steps which expect deployed artifacts and configuration to live under deployDir. Because logging is configured later via initLoggingConfg, the check uses a plain print to surface the problem to the operator, and the immediate exit ensures supervisors or CI see a failure instead of the app continuing in an invalid state. This differs from the nearby lines that simply assign or print deployDir and logFileDir: those are informational or configuration reads, whereas this is an explicit guard that enforces a required runtime precondition.

```python
# file path: src/main.py
if os.path.exists(logFileDir) == False:
  print("LogFile Directory " + logFileDir + " does not exist. Exiting the app.")
  exit(-1)

```

The entrypoint verifies that the filesystem path held in logFileDir actually exists by invoking os.path.exists and, if the directory is missing, emits a plain stdout message and terminates the process with a non-zero exit. This is a fail-fast precondition that runs before initLoggingConfg is called so the runtime never attempts to create or open the application log file inside a non-existent directory; because Python's logging system has not been configured yet the code uses print for the immediate error message. Compared to the earlier line that simply echoed the logFileDir value and the earlier assignment that populated logFileDir from serverConfig, this check enforces that the logging sink can be created and forces the application to stop cleanly when that prerequisite is absent.

```python
# file path: src/config/Config.py
def getSystemConfig():
  with open('../config/system.json', 'r') as system:
    jsonSystemData = json.load(system)
    return jsonSystemData
```

getSystemConfig is the small synchronous helper in the centralized configuration loader that opens the system-level JSON configuration, parses it, and returns the resulting Python dictionary for callers to consume. Its purpose is to expose runtime-level

```python
# file path: src/config/Config.py
def getHolidays():
  with open('../config/holidays.json', 'r') as holidays:
    holidaysData = json.load(holidays)
    return holidaysData
```

As we saw earlier when getServerConfig and getBrokerAppConfig load startup settings and boot the logging sink, getHolidays plays the equivalent role for calendar data: it performs a simple file I/O read of the project's holidays configuration and returns the parsed JSON object so the rest of the runtime can consult official market holidays. Utilities in Utils — for example the functions that generate expiry symbols and the holiday-checking helpers used by strategies and the scheduler — call getHolidays to determine whether a given date is a holiday, to skip trading days, and to drive expiry calculations. Like getServerConfig and getSystemConfig, getHolidays follows the same lightweight pattern of reading a configuration file and returning the deserialized data; unlike getTimestampsData it does not perform deploy-directory resolution or missing-file handling, it simply reads and returns the holidays payload for callers to use.

```python
# file path: src/config/Config.py
def getTimestampsData():
  serverConfig = getServerConfig()
  timestampsFilePath = os.path.join(serverConfig['deployDir'], 'timestamps.json')
  if os.path.exists(timestampsFilePath) == False:
    return {}
  timestampsFile = open(timestampsFilePath, 'r')
  timestamps = json.loads(timestampsFile.read())
  return timestamps
```

getTimestampsData uses the already-familiar getServerConfig to locate the runtime's deploy directory, then looks for a timestamps.json file inside that deployDir; if the file is missing it returns an empty dictionary, and if the file exists it opens and parses the JSON and returns the resulting Python dictionary. Conceptually this is the read side of the timestamps persistence pair that complements saveTimestampsData: startup components like Instruments.fetchInstrumentsFromServer and TradeManager.registerStrategy call getTimestampsData to learn the last-saved timestamps so they can decide whether to fetch fresh instrument data or resume work without duplicating work. The function performs a simple existence check before doing file I/O so callers get a safe empty state when no persisted timestamps are present, and it relies on getServerConfig to centralize where deployment artifacts (including timestamps.json) live.

```python
# file path: src/config/Config.py
def saveTimestampsData(timestamps = {}):
  serverConfig = getServerConfig()
  timestampsFilePath = os.path.join(serverConfig['deployDir'], 'timestamps.json')
  with open(timestampsFilePath, 'w') as timestampsFile:
    json.dump(timestamps, timestampsFile, indent=2)
  print("saved timestamps data to file " + timestampsFilePath)
```

saveTimestampsData accepts a timestamps dictionary and persists that runtime metadata as a JSON file inside the deployment directory defined by the server configuration. It first obtains serverConfig by calling getServerConfig, uses the deployDir value from that configuration to form the path for a timestamps.json file, opens that file for writing and writes the supplied timestamps object as pretty-printed JSON, and then prints a short console message confirming where the file was written. Conceptually this is the write-side companion to getTimestampsData: components like Instruments.fetchInstrumentsFromServer and TradeManager.registerStrategy update or create timestamp entries at runtime and call saveTimestampsData so authentication state, last-instrument-fetch times, and other cross-module timestamps survive process restarts and can be read back by authentication, controller, and instrument-loading logic. The observable side effects are file I/O to the deployDir and a console print for operator feedback.

```python
# file path: src/strategies/BaseStrategy.py
import logging
import time
from datetime import datetime
from models.ProductType import ProductType
from core.Quotes import Quotes
from trademgmt.TradeManager import TradeManager
from utils.Utils import Utils
```

BaseStrategy pulls together a small set of runtime, domain and helper modules so derived strategies can read market data, classify orders, interact with the execution layer, and emit lifecycle diagnostics. The logging module provides the process-wide logger that was configured earlier during startup so strategy events, decisions and errors are written to the established application log. time and datetime supply the lightweight timing and timestamp utilities the strategy uses for scheduling, measuring durations and tagging events as the strategy lifecycle advances. ProductType is the domain model enum that lets BaseStrategy label orders and positions with the correct product classification (for example differentiating intraday versus delivery), which the TradeManager and broker adapters rely on. Quotes is the normalized market data accessor that BaseStrategy uses to fetch the current price/quote view for instruments so strategy logic can make decisions without dealing with broker-specific payloads. TradeManager is the trade lifecycle and execution coordinator that BaseStrategy uses to place, modify, cancel and query orders and to reconcile trade state with the strategy’s internal bookkeeping. Utils supplies shared helper routines (formatting, calculations, small conversions) that keep repeated utility logic out of strategy code. This import set mirrors the common pattern you’ve seen elsewhere in the codebase: most strategy-adjacent modules import logging and datetime for observability, ProductType and trade-management primitives for execution semantics, and a small shared Utils library, while other files add Direction or Instruments when they need order polarity or instrument lookup functionality.

```python
# file path: src/strategies/BaseStrategy.py
class BaseStrategy:
  def __init__(self, name):
    self.name = name 
    self.enabled = True 
    self.productType = ProductType.MIS 
    self.symbols = [] 
    self.slPercentage = 0
    self.targetPercentage = 0
    self.startTimestamp = Utils.getMarketStartTime() 
    self.stopTimestamp = None 
    self.squareOffTimestamp = None 
    self.capital = 10000 
    self.leverage = 1 
    self.maxTradesPerDay = 1 
    self.isFnO = False 
    self.capitalPerSet = 0 
    TradeManager.registerStrategy(self)
    self.trades = TradeManager.getAllTradesByStrategy(self.name)
  def getName(self):
    return self.name
  def isEnabled(self):
    return self.enabled
  def setDisabled(self):
    self.enabled = False
  def process(self):
    logging.info("BaseStrategy process is called.")
    pass
  def calculateCapitalPerTrade(self):
    leverage = self.leverage if self.leverage > 0 else 1
    capitalPerTrade = int(self.capital * leverage / self.maxTradesPerDay)
    return capitalPerTrade
  def calculateLotsPerTrade(self):
    if self.isFnO == False:
      return 0
    return int(self.capital / self.capitalPerSet)
  def canTradeToday(self):
    return True
  def run(self):
    if self.enabled == False:
      logging.warn("%s: Not going to run strategy as its not enabled.", self.getName())
      return
    if Utils.isMarketClosedForTheDay():
      logging.warn("%s: Not going to run strategy as market is closed.", self.getName())
      return
    now = datetime.now()
    if now < Utils.getMarketStartTime():
      Utils.waitTillMarketOpens(self.getName())
    if self.canTradeToday() == False:
      logging.warn("%s: Not going to run strategy as it cannot be traded today.", self.getName())
      return
    now = datetime.now()
    if now < self.startTimestamp:
      waitSeconds = Utils.getEpoch(self.startTimestamp) - Utils.getEpoch(now)
      logging.info("%s: Waiting for %d seconds till startegy start timestamp reaches...", self.getName(), waitSeconds)
      if waitSeconds > 0:
        time.sleep(waitSeconds)      
    while True:
      if Utils.isMarketClosedForTheDay():
        logging.warn("%s: Exiting the strategy as market closed.", self.getName())
        break
```

BaseStrategy defines the common lifecycle, defaults and small helpers that every concrete strategy builds on, and it wires each strategy instance into TradeManager so the engine can track and persist strategy-specific trades. On construction BaseStrategy captures a name and sets a set of sensible defaults for enablement, product type, symbols list, stop/target percentages, start/stop/square-off timestamps, capital, leverage, max trades per day, F&O flags and capital-per-set; it then registers the instance with TradeManager.registerStrategy and populates its trades list by calling TradeManager.getAllTradesByStrategy so any previously persisted intraday trades are available to the strategy. It exposes simple identity and enablement accessors (getName, isEnabled, setDisabled) and a process placeholder that logs invocation and is intended to be overridden by derived classes such as BNFORB30Min, OptionSelling and SampleStrategy. Two utility calculators are provided: calculateCapitalPerTrade computes an integer allocation per trade using leverage and the configured maxTradesPerDay, and calculateLotsPerTrade returns zero for non-FnO strategies or derives lot counts from capital and capitalPerSet for FnO. canTradeToday returns true by default so strategies can override business-day logic. The run method orchestrates the strategy’s runtime gating: it skips execution when disabled, consults market state via Utils.isMarketClosedForTheDay and uses Utils.waitTillMarketOpens and Utils.getMarketStartTime to defer until market open, checks canTradeToday, sleeps until the configured startTimestamp using Utils.getEpoch when necessary, and then enters a loop that exits once Utils reports the market is closed. The class therefore centralizes the repeated preconditions and scheduling behavior that concrete strategies reuse while leaving trade generation, placement and per-tick decision methods to the subclasses and to the rest of the TradeManager/Utils ecosystem.

```python
# file path: src/utils/Utils.py
import math
import uuid
import time
import logging
import calendar
from datetime import datetime, timedelta
from config.Config import getHolidays
from models.Direction import Direction
from trademgmt.TradeState import TradeState
```

The imports here bring together a small set of standard library utilities and a couple of domain types that Utilities rely on across the framework: math supplies numeric helpers for calculations used in normalization and risk math, uuid is used when utilities need to create unique identifiers for ephemeral objects or test fixtures, and time plus datetime and timedelta provide the runtime timekeeping and arithmetic needed for timestamping, interval calculations and market session logic; calendar is pulled in to assist with weekday/holiday computations that are combined with the project's holiday data from getHolidays (you’ve already seen getHolidays used elsewhere to load the market calendar). Logging is imported so utility functions can emit diagnostics consistent with the rest of the system. Finally, Direction and TradeState are the domain enums the utilities use to interpret and normalize trade-side semantics and lifecycle state when helpers process trades or build normalized quote/position views. Compared to the similar import lists found in strategy and core modules—which often import ProductType, Quotes, TradeManager or the central Utils class—this file focuses on foundational, low-level helpers (stdlib time/math/uuid/calendar) plus the minimal domain types (Direction, TradeState) and holiday loader needed to keep shared normalization and date logic consistent across strategies and tests.

```python
# file path: src/utils/Utils.py
class Utils:
  dateFormat = "%Y-%m-%d"
  timeFormat = "%H:%M:%S"
  dateTimeFormat = "%Y-%m-%d %H:%M:%S"
  @staticmethod
  def roundOff(price): 
    return round(price, 2)
  @staticmethod
  def roundToNSEPrice(price):
    x = round(price, 2) * 20
    y = math.ceil(x)
    return y / 20
  @staticmethod
  def isMarketOpen():
    if Utils.isTodayHoliday():
      return False
    now = datetime.now()
    marketStartTime = Utils.getMarketStartTime()
    marketEndTime = Utils.getMarketEndTime()
    return now >= marketStartTime and now <= marketEndTime
  @staticmethod
  def isMarketClosedForTheDay():
    if Utils.isTodayHoliday():
      return True
    now = datetime.now()
    marketEndTime = Utils.getMarketEndTime()
    return now > marketEndTime
  @staticmethod
  def waitTillMarketOpens(context):
    nowEpoch = Utils.getEpoch(datetime.now())
    marketStartTimeEpoch = Utils.getEpoch(Utils.getMarketStartTime())
    waitSeconds = marketStartTimeEpoch - nowEpoch
    if waitSeconds > 0:
      logging.info("%s: Waiting for %d seconds till market opens...", context, waitSeconds)
      time.sleep(waitSeconds)
  @staticmethod
  def getEpoch(datetimeObj = None):
    if datetimeObj == None:
      datetimeObj = datetime.now()
    epochSeconds = datetime.timestamp(datetimeObj)
    return int(epochSeconds) 
  @staticmethod
  def getMarketStartTime(dateTimeObj = None):
    return Utils.getTimeOfDay(9, 15, 0, dateTimeObj)
  @staticmethod
  def getMarketEndTime(dateTimeObj = None):
    return Utils.getTimeOfDay(15, 30, 0, dateTimeObj)
  @staticmethod
  def getTimeOfDay(hours, minutes, seconds, dateTimeObj = None):
    if dateTimeObj == None:
      dateTimeObj = datetime.now()
    dateTimeObj = dateTimeObj.replace(hour=hours, minute=minutes, second=seconds, microsecond=0)
    return dateTimeObj
  @staticmethod
  def getTimeOfToDay(hours, minutes, seconds):
    return Utils.getTimeOfDay(hours, minutes, seconds, datetime.now())
  @staticmethod
  def getTodayDateStr():
    return Utils.convertToDateStr(datetime.now())
  @staticmethod
```

Utils centralizes small, commonly used helpers around numeric rounding and market-time calculations that other mid-level modules and strategies rely on to make time- and price-based decisions. It defines the standard date, time and datetime formats used across the project and provides two rounding helpers: roundOff performs a simple two-decimal currency-style rounding used by P&L and price reporting, and roundToNSEPrice snaps an arbitrary price up to the next valid NSE tick by scaling, applying a ceiling and rescaling (effectively enforcing a 0.05 tick granularity). The market-status helpers drive the runtime behaviour of strategies and managers: isMarketOpen first consults the holiday logic (via the holiday helper in the adjacent Utils part) and then compares the current instant to market window boundaries; isMarketClosedForTheDay flags that the remainder of the day is non-tradable either because of a holiday or because the current time is past the market close. waitTillMarketOpens computes seconds until the next market start and sleeps while emitting an informational log using the provided context string; it relies on getEpoch to convert datetimes to integer epoch seconds. getMarketStartTime and getMarketEndTime are convenience accessors that produce datetimes set to the official market open and close times by delegating to getTimeOfDay, which normalizes any passed datetime (or now) to a specific hour/minute/second with microseconds cleared. getTimeOfToDay is a thin convenience wrapper for constructing those day-specific datetimes for the current day, and getTodayDateStr

```python
# file path: src/utils/Utils.py
  def convertToDateStr(datetimeObj):
    return datetimeObj.strftime(Utils.dateFormat)
  @staticmethod
  def isHoliday(datetimeObj):
    dayOfWeek = calendar.day_name[datetimeObj.weekday()]
    if dayOfWeek == 'Saturday' or dayOfWeek == 'Sunday':
      return True
    dateStr = Utils.convertToDateStr(datetimeObj)
    holidays = getHolidays()
    if (dateStr in holidays):
      return True
    else:
      return False
  @staticmethod
  def isTodayHoliday():
    return Utils.isHoliday(datetime.now())
  @staticmethod
  def generateTradeID():
    return str(uuid.uuid4())
  @staticmethod
  def calculateTradePnl(trade):
    if trade.tradeState == TradeState.ACTIVE:
      if trade.cmp > 0:
        if trade.direction == Direction.LONG:
          trade.pnl = Utils.roundOff(trade.filledQty * (trade.cmp - trade.entry))
        else:  
          trade.pnl = Utils.roundOff(trade.filledQty * (trade.entry - trade.cmp))
    else:
      if trade.exit > 0:
        if trade.direction == Direction.LONG:
          trade.pnl = Utils.roundOff(trade.filledQty * (trade.exit - trade.entry))
        else:  
          trade.pnl = Utils.roundOff(trade.filledQty * (trade.entry - trade.exit))
    tradeValue = trade.entry * trade.filledQty
    if tradeValue > 0:
      trade.pnlPercentage = Utils.roundOff(trade.pnl * 100 / tradeValue)
    return trade
  @staticmethod
  def prepareMonthlyExpiryFuturesSymbol(inputSymbol):
    expiryDateTime = Utils.getMonthlyExpiryDayDate()
    expiryDateMarketEndTime = Utils.getMarketEndTime(expiryDateTime)
    now = datetime.now()
    if now > expiryDateMarketEndTime:
      expiryDateTime = Utils.getMonthlyExpiryDayDate(now + timedelta(days=20))
    year2Digits = str(expiryDateTime.year)[2:]
    monthShort = calendar.month_name[expiryDateTime.month].upper()[0:3]
    futureSymbol = inputSymbol + year2Digits + monthShort + 'FUT'
    logging.info('prepareMonthlyExpiryFuturesSymbol[%s] = %s', inputSymbol, futureSymbol)  
    return futureSymbol
  @staticmethod
  def prepareWeeklyOptionsSymbol(inputSymbol, strike, optionType, numWeeksPlus = 0):
    expiryDateTime = Utils.getWeeklyExpiryDayDate()
    todayMarketStartTime = Utils.getMarketStartTime()
    expiryDayMarketEndTime = Utils.getMarketEndTime(expiryDateTime)
    if numWeeksPlus > 0:
      expiryDateTime = expiryDateTime + timedelta(days=numWeeksPlus * 7)
      expiryDateTime = Utils.getWeeklyExpiryDayDate(expiryDateTime)
    if todayMarketStartTime > expiryDayMarketEndTime:
      expiryDateTime = expiryDateTime + timedelta(days=6)
      expiryDateTime = Utils.getWeeklyExpiryDayDate(expiryDateTime)
```

convertToDateStr is a simple formatter that turns a Python datetime into the project's canonical date string using the dateFormat constant defined on Utils in Utils_part1, so downstream code always compares dates in the same textual form. isHoliday performs the project's holiday logic for any datetime: it first rejects weekends by checking the weekday name, then converts the datetime to the canonical date string via convertToDateStr and consults the holidays list returned by getHolidays (which you already saw reads the holidays.json file); if either condition matches it reports the day as a holiday. isTodayHoliday is just a convenience wrapper that asks isHoliday about the current moment. generateTradeID returns a unique identifier for a new Trade by producing a UUID4 string, which Trade uses when instantiated. calculateTradePnl accepts a Trade object and computes and assigns its running PnL and PnL percentage: when the trade is active it compares the current market price against entry depending on the trade direction, and when the trade has exited it uses the recorded exit price vs entry; numeric results are normalized with Utils.roundOff and the percent is computed against the notional entry value. prepareMonthlyExpiryFuturesSymbol builds a monthly futures symbol for a given underlying by first finding the monthly expiry date via Utils.getMonthlyExpiryDayDate (logic for which lives in Utils_part3), then checking whether the current time is beyond that expiry's market end and, if so, advancing to the next month before formatting the symbol; it forms the suffix from the two-digit year and the three-letter uppercase month name and appends the futures marker, and emits a log entry with the produced symbol. prepareWeeklyOptionsSymbol begins the weekly-option symbol workflow by resolving the appropriate weekly expiry date using Utils.getWeeklyExpiryDayDate (from Utils_part3), compares market start and expiry market end times to decide whether to roll the expiry forward (and supports an explicit numWeeksPlus offset by shifting weeks and re-resolving the weekly expiry), and then proceeds (in the adjacent code in Utils_part3) to construct the option symbol string in the appropriate monthly-or-day-encoded format; this function therefore orchestrates expiry selection and defers the final symbol-encoding details to the companion logic in Utils_part3.

```python
# file path: src/utils/Utils.py
    expiryDateTimeMonthly = Utils.getMonthlyExpiryDayDate()
    weekAndMonthExpriySame = False
    if expiryDateTime == expiryDateTimeMonthly:
      weekAndMonthExpriySame = True
      logging.info('Weekly and Monthly expiry is same for %s', expiryDateTime)
    year2Digits = str(expiryDateTime.year)[2:]
    optionSymbol = None
    if weekAndMonthExpriySame == True:
      monthShort = calendar.month_name[expiryDateTime.month].upper()[0:3]
      optionSymbol = inputSymbol + str(year2Digits) + monthShort + str(strike) + optionType.upper()
    else:
      m = expiryDateTime.month
      d = expiryDateTime.day
      mStr = str(m)
      if m == 10:
        mStr = "O"
      elif m == 11:
        mStr = "N"
      elif m == 12:
        mStr = "D"
      dStr = ("0" + str(d)) if d < 10 else str(d)
      optionSymbol = inputSymbol + str(year2Digits) + mStr + dStr + str(strike) + optionType.upper()
    logging.info('prepareWeeklyOptionsSymbol[%s, %d, %s, %d] = %s', inputSymbol, strike, optionType, numWeeksPlus, optionSymbol)  
    return optionSymbol
  @staticmethod
  def getMonthlyExpiryDayDate(datetimeObj = None):
    if datetimeObj == None:
      datetimeObj = datetime.now()
    year = datetimeObj.year
    month = datetimeObj.month
    lastDay = calendar.monthrange(year, month)[1] 
    datetimeExpiryDay = datetime(year, month, lastDay)
    while calendar.day_name[datetimeExpiryDay.weekday()] != 'Thursday':
      datetimeExpiryDay = datetimeExpiryDay - timedelta(days=1)
    while Utils.isHoliday(datetimeExpiryDay) == True:
      datetimeExpiryDay = datetimeExpiryDay - timedelta(days=1)
    datetimeExpiryDay = Utils.getTimeOfDay(0, 0, 0, datetimeExpiryDay)
    return datetimeExpiryDay
  @staticmethod
  def getWeeklyExpiryDayDate(dateTimeObj = None):
    if dateTimeObj == None:
      dateTimeObj = datetime.now()
    daysToAdd = 0
    if dateTimeObj.weekday() >= 3:
      daysToAdd = -1 * (dateTimeObj.weekday() - 3)
    else:
      daysToAdd = 3 - dateTimeObj.weekday()
    datetimeExpiryDay = dateTimeObj + timedelta(days=daysToAdd)
    while Utils.isHoliday(datetimeExpiryDay) == True:
      datetimeExpiryDay = datetimeExpiryDay - timedelta(days=1)
    datetimeExpiryDay = Utils.getTimeOfDay(0, 0, 0, datetimeExpiryDay)
    return datetimeExpiryDay
  @staticmethod
  def isTodayWeeklyExpiryDay():
    expiryDate = Utils.getWeeklyExpiryDayDate()
    todayDate = Utils.getTimeOfToDay(0, 0, 0)
    if expiryDate == todayDate:
      return True
    return False
  @staticmethod
```

Utils_part3 provides the holiday-aware expiry date calculations and the branching logic used to build option symbols that other strategy modules rely on when they need to reference weekly or monthly expiries. The prepareWeeklyOptionsSymbol logic first asks Utils.getMonthlyExpiryDayDate for the month's expiry and compares it to the weekly expiry it was already computing; if the weekly and monthly expiries coincide it formats the options trading symbol using the two-digit year plus a three-letter uppercase month abbreviation, strike and option side, otherwise it formats the symbol using a compact encoding (

```python
# file path: src/trademgmt/TradeManager.py
import os
import logging
import time
import json
from datetime import datetime
from config.Config import getServerConfig
from core.Controller import Controller
from ticker.ZerodhaTicker import ZerodhaTicker
from trademgmt.Trade import Trade
from trademgmt.TradeState import TradeState
from trademgmt.TradeExitReason import TradeExitReason
from trademgmt.TradeEncoder import TradeEncoder
from ordermgmt.ZerodhaOrderManager import ZerodhaOrderManager
from ordermgmt.OrderInputParams import OrderInputParams
from ordermgmt.OrderModifyParams import OrderModifyParams
from ordermgmt.Order import Order
from models.OrderType import OrderType
from models.OrderStatus import OrderStatus
from models.Direction import Direction
from utils.Utils import Utils
```

TradeManager pulls together a small set of runtime, domain and broker-specific building blocks so it can translate strategy signals into executable orders and durable trade records. The standard Python imports os, logging, time, json and datetime provide filesystem access, runtime diagnostics, simple timing, JSON serialization and timestamping needed for persisting trade state and measuring lifecycle events. getServerConfig is used to locate runtime configuration and deploy paths (recall getServerConfig from earlier), while Controller gives access to the core runtime context that TradeManager uses to interact with other engine components. ZerodhaTicker and ZerodhaOrderManager are the broker-specific adapters for market data and execution that TradeManager will call into when it needs live quotes or to place/modify/cancel orders. The trade domain is represented by Trade, TradeState, TradeExitReason and TradeEncoder so TradeManager can create trade objects, record state transitions and serialize them for storage or messaging. OrderInputParams and OrderModifyParams are the parameter wrappers TradeManager constructs when building new orders or issuing modifications, and Order along with OrderType, OrderStatus and Direction are the shared order-model enums and structures used throughout the engine to track the lifecycle of individual execution requests. Finally, Utils supplies common helpers (formatting, rounding, time helpers) that TradeManager uses for small transformations. This import set follows the same project pattern seen elsewhere—reusing logging, order models and Utils across modules—but differs from the other import blocks by combining configuration and controller access with the trade lifecycle classes and broker-specific order/ticker adapters, reflecting TradeManager’s role as the glue between strategy signals, broker execution and persistent trade bookkeeping.

```python
# file path: src/trademgmt/TradeManager.py
class TradeManager:
  ticker = None
  trades = [] 
  strategyToInstanceMap = {}
  symbolToCMPMap = {}
  intradayTradesDir = None
  registeredSymbols = []
  @staticmethod
  def run():
    if Utils.isTodayHoliday():
      logging.info("Cannot start TradeManager as Today is Trading Holiday.")
      return
    if Utils.isMarketClosedForTheDay():
      logging.info("Cannot start TradeManager as Market is closed for the day.")
      return
    Utils.waitTillMarketOpens("TradeManager")
    serverConfig = getServerConfig()
    tradesDir = os.path.join(serverConfig['deployDir'], 'trades')
    TradeManager.intradayTradesDir =  os.path.join(tradesDir, Utils.getTodayDateStr())
    if os.path.exists(TradeManager.intradayTradesDir) == False:
      logging.info('TradeManager: Intraday Trades Directory %s does not exist. Hence going to create.', TradeManager.intradayTradesDir)
      os.makedirs(TradeManager.intradayTradesDir)
    brokerName = Controller.getBrokerName()
    if brokerName == "zerodha":
      TradeManager.ticker = ZerodhaTicker()
    TradeManager.ticker.startTicker()
    TradeManager.ticker.registerListener(TradeManager.tickerListener)
    time.sleep(2)
    TradeManager.loadAllTradesFromFile()
    while True:
      if Utils.isMarketClosedForTheDay():
        logging.info('TradeManager: Stopping TradeManager as market closed.')
        break
      try:
        TradeManager.fetchAndUpdateAllTradeOrders()
        TradeManager.trackAndUpdateAllTrades()
      except Exception as e:
        logging.exception("Exception in TradeManager Main thread")
      TradeManager.saveAllTradesToFile()
      time.sleep(30)
      logging.info('TradeManager: Main thread woke up..')
  @staticmethod
  def registerStrategy(strategyInstance):
    TradeManager.strategyToInstanceMap[strategyInstance.getName()] = strategyInstance
  @staticmethod
  def loadAllTradesFromFile():
    tradesFilepath = os.path.join(TradeManager.intradayTradesDir, 'trades.json')
    if os.path.exists(tradesFilepath) == False:
      logging.warn('TradeManager: loadAllTradesFromFile() Trades Filepath %s does not exist', tradesFilepath)
      return
    TradeManager.trades = []
    tFile = open(tradesFilepath, 'r')
    tradesData = json.loads(tFile.read())
    for tr in tradesData:
      trade = TradeManager.convertJSONToTrade(tr)
      logging.info('loadAllTradesFromFile trade => %s', trade)
      TradeManager.trades.append(trade)
      if trade.tradingSymbol not in TradeManager.registeredSymbols:
        TradeManager.ticker.registerSymbols([trade.tradingSymbol])
        TradeManager.registeredSymbols.append(trade.tradingSymbol)
```

TradeManager implements the lifecycle controller that glues strategies to execution: the TradeManager class holds runtime state (a ticker instance, the in-memory list of Trade objects, a map from strategy name to strategy instance, a symbol-to-current-market-price map, the intraday trades directory path, and a list of symbols already registered with the market feed). The run method is the orchestrator that first consults Utils to skip startup on market holidays or a closed market and then waits until market open when needed; it then loads the server configuration via getServerConfig to derive a deploy-level trades directory and builds an intraday folder named for today, creating it if absent. After discovering the broker via Controller.getBrokerName it instantiates the appropriate ticker adaptor (the code uses ZerodhaTicker for the zerodha broker), starts the ticker and registers TradeManager.tickerListener as a tick consumer, sleeps briefly to let the feed stabilize, and then calls loadAllTradesFromFile to hydrate any persisted trades for today. loadAllTradesFromFile looks for a trades.json under the intraday directory, logs and returns if missing, otherwise clears the in-memory trades list, parses the JSON, converts each JSON object into a Trade using convertJSONToTrade, appends them to TradeManager.trades, logs each load and registers the trade’s tradingSymbol with the ticker if that symbol hasn’t already been registered. After initialization run enters a loop that stops once Utils reports the market closed; each cycle it calls fetchAndUpdateAllTradeOrders and trackAndUpdateAllTrades to reconcile live order state and advance trade state (those behaviors are implemented in other TradeManager parts), catches and logs exceptions from the cycle, persists the current trades via saveAllTradesToFile, then sleeps for a fixed interval before repeating. The registerStrategy helper simply records a strategy instance into strategyToInstanceMap by its BaseStrategy name so incoming ticks and lifecycle actions can route to the correct strategy.

```python
# file path: src/ticker/BaseTicker.py
import logging
from core.Controller import Controller
```

It imports Python's logging so BaseTicker can emit structured runtime diagnostics for tick processing, error conditions, and lifecycle events, and it imports Controller from core.Controller so BaseTicker can delegate decision-making and lifecycle control to a Controller instance rather than embedding that logic itself. In the project's architecture, that mirrors the separation of concerns: BaseTicker provides the feed, event handling, and lifecycle hooks while Controller encapsulates the trading control logic. Similar import patterns elsewhere show the same split—some modules import Controller alongside domain models like Quote when they need both control logic and data objects, while broker-adaptors and startup code import configuration helpers and broker-specific login classes such as getBrokerAppConfig, BrokerAppDetails, or ZerodhaLogin when they must perform connectivity and authentication. This file keeps its imports minimal because its role is to standardize tick flow and controller integration rather than handle broker-specific or model-level responsibilities.

```python
# file path: src/ticker/BaseTicker.py
class BaseTicker:
  def __init__(self, broker):
    self.broker = broker
    self.brokerLogin = Controller.getBrokerLogin()
    self.ticker = None
    self.tickListeners = []
  def startTicker(self):
    pass
  def stopTicker(self):
    pass
  def registerListener(self, listener):
    self.tickListeners.append(listener)
  def registerSymbols(self, symbols):
    pass
  def unregisterSymbols(self, symbols):
    pass
  def onNewTicks(self, ticks):
    for tick in ticks:
      for listener in self.tickListeners:
        try:
          listener(tick)
        except Exception as e:
          logging.error('BaseTicker: Exception from listener callback function. Error => %s', str(e))
  def onConnect(self):
    logging.info('Ticker connection successful.')
  def onDisconnect(self, code, reason):
    logging.error('Ticker got disconnected. code = %d, reason = %s', code, reason)
  def onError(self, code, reason):
    logging.error('Ticker errored out. code = %d, reason = %s', code, reason)
  def onReconnect(self, attemptsCount):
    logging.warn('Ticker reconnecting.. attemptsCount = %d', attemptsCount)
  def onMaxReconnectsAttempt(self):
    logging.error('Ticker max auto reconnects attempted and giving up..')
  def onOrderUpdate(self, data):
    pass
```

BaseTicker provides the common runtime contract and shared behavior that lets TradeManager, Test and broker-specific tickers plug into a single, predictable tick delivery pipeline so the rest of the algo framework can treat different broker adapters uniformly. On construction BaseTicker records the broker identifier you passed and grabs the shared brokerLogin object from Controller.getBrokerLogin so every ticker instance starts with the same authenticated context; it also initializes an internal tickListeners list that other components register callbacks into. The startTicker, stopTicker, registerSymbols, unregisterSymbols and onOrderUpdate methods are defined as no-ops here because concrete tickers such as ZerodhaTicker implement the actual connection, subscription and order-update behavior; ZerodhaTicker, for example, uses the brokerLogin to obtain app credentials and access tokens and wires its websocket callbacks before calling into BaseTicker’s delivery path. registerListener appends a listener callback to the internal list so consumers like TradeManager.tickerListener or Test.tickerListener can subscribe to live ticks. onNewTicks is the distributor: it accepts a sequence of normalized tick objects and iterates through each tick and each registered listener, invoking the callbacks inside a try/except and logging any listener exceptions so a faulty strategy callback does not collapse the feed. The connection lifecycle hooks onConnect, onDisconnect, onError, onReconnect and onMaxReconnectsAttempt provide consistent logging and a single place for runtime diagnostics; subclasses map their broker-specific websocket events onto these hooks (ZerodhaTicker maps its on_connect/on_close/on_error/on_reconnect/on_noreconnect to these). In short, BaseTicker centralizes listener management, standardized event logging and the handshake to Controller-provided authentication, while delegating transport, symbol-to-token mapping and tick normalization to broker-specific subclasses so the rest of the sdoosa-algo-trade-python-master_cleaned system can consume a uniform tick stream.

```python
# file path: src/core/Controller.py
import logging
from config.Config import getBrokerAppConfig
from models.BrokerAppDetails import BrokerAppDetails
from loginmgmt.ZerodhaLogin import ZerodhaLogin
```

Controller imports the standard logging facility so it can emit runtime diagnostics and integrate with the project's logging sink. It pulls getBrokerAppConfig from config.Config; analogous to the getSystemConfig helper you already saw, getBrokerAppConfig is the config accessor that returns the broker-specific runtime settings Controller needs to decide which credentials and adapter to instantiate. BrokerAppDetails is the model that encapsulates that broker application metadata and credential fields so Controller can pass a typed object around the rest of the system. ZerodhaLogin is the Zerodha-specific login/session manager that implements the authentication flow, session tokens and any broker-specific handshake logic. Together these imports let Controller load broker configuration, populate a BrokerAppDetails instance, and drive ZerodhaLogin to obtain an authenticated connection that Controller supplies to Quotes, BaseOrderManager, and Instruments_part1. This follows the common project pattern where modules import logging, a configuration getter and a login/adaptor class; other files use the same pattern but swap in getSystemConfig, KiteConnect or BaseLogin where appropriate, whereas Controller specifically uses getBrokerAppConfig and ZerodhaLogin because it centralizes Zerodha connection setup.

```python
# file path: src/core/Controller.py
class Controller:
  brokerLogin = None 
  brokerName = None 
  def handleBrokerLogin(args):
    brokerAppConfig = getBrokerAppConfig()
    brokerAppDetails = BrokerAppDetails(brokerAppConfig['broker'])
    brokerAppDetails.setClientID(brokerAppConfig['clientID'])
    brokerAppDetails.setAppKey(brokerAppConfig['appKey'])
    brokerAppDetails.setAppSecret(brokerAppConfig['appSecret'])
    logging.info('handleBrokerLogin appKey %s', brokerAppDetails.appKey)
    Controller.brokerName = brokerAppDetails.broker
    if Controller.brokerName == 'zerodha':
      Controller.brokerLogin = ZerodhaLogin(brokerAppDetails)
    redirectUrl = Controller.brokerLogin.login(args)
    return redirectUrl
  def getBrokerLogin():
    return Controller.brokerLogin
  def getBrokerName():
    return Controller.brokerName
```

Controller is the single orchestrator that centralizes broker configuration and authentication so the rest of the framework can obtain a single, shared broker session and its credentials. When an external login flow is triggered (for example by BrokerLoginAPI handing request arguments to the controller), handleBrokerLogin reads the broker application configuration from disk, creates a BrokerAppDetails instance and populates it with the configured client identifier, application key and secret, and records the broker name on a class-level attribute. It then selects and instantiates the broker-specific login implementation based on that broker name (for example, ZerodhaLogin when the broker is zerodha) and delegates the authentication flow to that login object; the broker login implementation performs the provider-specific session exchange, sets the broker handle and access token on the BaseLogin-derived instance, and returns a redirect URL which handleBrokerLogin forwards back to the caller. Controller exposes two simple accessors, getBrokerLogin and getBrokerName, that return the stored broker-login instance and the broker identifier respectively; other parts of the system call those accessors (BaseOrderManager to obtain the broker handle for order placement, BaseTicker to attach market feeds, API endpoints for holdings and positions, Instruments for instrument fetches, and TradeManager/strategies for runtime needs) so a single authenticated session and configuration are reused across the application. The design uses class-level brokerLogin and brokerName as a singleton-like shared state so the authentication result and broker details become the single source of truth for the rest of the modules.

```python
# file path: src/models/BrokerAppDetails.py
class BrokerAppDetails:
  def __init__(self, broker):
    self.broker = broker
    self.appKey = None
    self.appSecret = None

  def setClientID(self, clientID):
    self.clientID = clientID

  def setAppKey(self, appKey):
    self.appKey = appKey

  def setAppSecret(self, appSecret):
    self.appSecret = appSecret
```

BrokerAppDetails is a small value-object that holds a broker identifier and the credentials the Controller needs to construct an authenticator and broker session. Controller.handleBrokerLogin loads the broker configuration via getBrokerAppConfig and then constructs a BrokerAppDetails with the broker name; it then calls setClientID, setAppKey and setAppSecret to populate the instance. Those attributes — broker, clientID, appKey and appSecret — are the pieces ZerodhaLogin (which extends BaseLogin) will read through the BaseLogin pathway when establishing a broker handle and access token, and they are what other parts of the system rely on when the Controller supplies broker-specific parameters for market data, order execution and trade lifecycle management. The methods on BrokerAppDetails are simple setters that write those instance attributes; its role is purely to encapsulate and carry the broker app configuration from the Controller into the login and broker-adapter layers.

```python
# file path: src/loginmgmt/ZerodhaLogin.py
import logging
from kiteconnect import KiteConnect
from config.Config import getSystemConfig
from loginmgmt.BaseLogin import BaseLogin
```

ZerodhaLogin pulls in the runtime diagnostics facility via logging so it can emit the same structured runtime and error messages other modules already use; it brings in KiteConnect from the kiteconnect SDK because this adapter needs the broker-specific client to perform API key/secret based authentication, generate request/session URLs, and exchange tokens with Zerodha during login and session management. It reads environment and deployment values through getSystemConfig so the adapter can load API credentials, session persistence paths, and other system-level settings rather than hard-coding them into the class. Finally, ZerodhaLogin subclasses BaseLogin so it can implement the broker-specific parts of the shared login contract and reuse common login/session lifecycle behavior defined by the framework. This import set follows the same pattern seen elsewhere—logging is reused across modules, but where other files used KiteTicker for live ticks, ZerodhaLogin uses KiteConnect for the authentication handshake, and where some modules pulled getBrokerAppConfig for app-specific settings, ZerodhaLogin uses getSystemConfig to obtain broader system-level credentials needed at startup.

```python
# file path: src/loginmgmt/ZerodhaLogin.py
class ZerodhaLogin(BaseLogin):
  def __init__(self, brokerAppDetails):
    BaseLogin.__init__(self, brokerAppDetails)
  def login(self, args):
    logging.info('==> ZerodhaLogin .args => %s', args);
    systemConfig = getSystemConfig()
    brokerHandle = KiteConnect(api_key=self.brokerAppDetails.appKey)
    redirectUrl = None
    if 'request_token' in args:
      requestToken = args['request_token']
      logging.info('Zerodha requestToken = %s', requestToken)
      session = brokerHandle.generate_session(requestToken, api_secret=self.brokerAppDetails.appSecret)
      accessToken = session['access_token']
      accessToken = accessToken
      logging.info('Zerodha accessToken = %s', accessToken)
      brokerHandle.set_access_token(accessToken)
      logging.info('Zerodha Login successful. accessToken = %s', accessToken)
      self.setBrokerHandle(brokerHandle)
      self.setAccessToken(accessToken)
      homeUrl = systemConfig['homeUrl'] + '?loggedIn=true'
      logging.info('Zerodha Redirecting to home page %s', homeUrl)
      redirectUrl = homeUrl
    else:
      loginUrl = brokerHandle.login_url()
      logging.info('Redirecting to zerodha login url = %s', loginUrl)
      redirectUrl = loginUrl
    return redirectUrl
```

ZerodhaLogin extends BaseLogin and implements the Zerodha-specific authentication and session setup used by Controller to establish a live broker session. Its constructor delegates initialization to BaseLogin so the instance carries the BrokerAppDetails prepared by Controller.handleBrokerLogin. The login method accepts the incoming request arguments and first reads runtime configuration via getSystemConfig to later construct a post-login redirect. It then instantiates the KiteConnect client using the app key from BrokerAppDetails. If the incoming arguments contain a request_token (the OAuth callback path after the user has authenticated with Zerodha), ZerodhaLogin exchanges that request token for a session using the app secret, extracts the access token from the returned session, configures the KiteConnect client with that access token, and persists the KiteConnect handle and access token onto the BaseLogin instance by calling setBrokerHandle and setAccessToken so Controller.getBrokerLogin can expose an authenticated brokerHandle to the rest of the system. After a successful exchange it builds a home URL using the system config and returns it as the redirect destination. If no request_token is present (the initial login flow), ZerodhaLogin asks KiteConnect for the provider login URL and returns that so the web endpoint can redirect the user to Zerodha’s authorization page. Throughout the method it emits structured logging. The net effect is that, once login returns an authenticated KiteConnect placed on BaseLogin, downstream components such as BaseOrderManager, HoldingsAPI, PositionsAPI, and the ticker plumbing can obtain an authenticated brokerHandle via Controller.getBrokerLogin and make authenticated API calls on behalf of the user.

```python
# file path: src/ticker/ZerodhaTicker.py
import logging
import json
from kiteconnect import KiteTicker
from ticker.BaseTicker import BaseTicker
from instruments.Instruments import Instruments
from models.TickData import TickData
```

The file pulls together the small set of building blocks the Zerodha-specific ticker needs to implement real-time market feed behavior: logging is imported so ZerodhaTicker can emit the same structured runtime diagnostics and lifecycle events you saw used elsewhere; json is available to parse or serialize raw messages coming from the streaming layer or to persist/inspect payloads; KiteTicker is the Kite Connect streaming client that provides the websocket-based market feed from Zerodha (this complements other files that use KiteConnect for REST operations, but here we need the live-tick interface); BaseTicker is brought in so ZerodhaTicker can implement the shared runtime contract and integrate cleanly with TradeManager and the rest of the framework; Instruments supplies the instrument metadata and token-to-symbol mapping logic ZerodhaTicker uses to translate broker tokens into framework symbols and to look up instrument details; TickData is the framework’s normalized tick model that ZerodhaTicker will populate for every incoming market update so the rest of the system (strategies, TradeManager, order managers) can consume a consistent tick shape. This set of imports follows the same project pattern you’ve already seen—modules typically combine logging plus a broker SDK client, the appropriate base interface, domain models, and helper metadata classes—while differing from the KiteConnect-using modules by specifically depending on the streaming client and the TickData normalization type needed for live tick ingestion.

```python
# file path: src/ticker/ZerodhaTicker.py
class ZerodhaTicker(BaseTicker):
  def __init__(self):
    super().__init__("zerodha")
  def startTicker(self):
    brokerAppDetails = self.brokerLogin.getBrokerAppDetails()
    accessToken = self.brokerLogin.getAccessToken()
    if accessToken == None:
      logging.error('ZerodhaTicker startTicker: Cannot start ticker as accessToken is empty')
      return
    ticker = KiteTicker(brokerAppDetails.appKey, accessToken)
    ticker.on_connect = self.on_connect
    ticker.on_close = self.on_close
    ticker.on_error = self.on_error
    ticker.on_reconnect = self.on_reconnect
    ticker.on_noreconnect = self.on_noreconnect
    ticker.on_ticks = self.on_ticks
    ticker.on_order_update = self.on_order_update
    logging.info('ZerodhaTicker: Going to connect..')
    self.ticker = ticker
    self.ticker.connect(threaded=True)
  def stopTicker(self):
    logging.info('ZerodhaTicker: stopping..')
    self.ticker.close(1000, "Manual close")
  def registerSymbols(self, symbols):
    tokens = []
    for symbol in symbols:
      isd = Instruments.getInstrumentDataBySymbol(symbol)
      token = isd['instrument_token']
      logging.info('ZerodhaTicker registerSymbol: %s token = %s', symbol, token)
      tokens.append(token)
    logging.info('ZerodhaTicker Subscribing tokens %s', tokens)
    self.ticker.subscribe(tokens)
  def unregisterSymbols(self, symbols):
    tokens = []
    for symbol in symbols:
      isd = Instruments.getInstrumentDataBySymbol(symbol)
      token = isd['instrument_token']
      logging.info('ZerodhaTicker unregisterSymbols: %s token = %s', symbol, token)
      tokens.append(token)
    logging.info('ZerodhaTicker Unsubscribing tokens %s', tokens)
    self.ticker.unsubscribe(tokens)
  def on_ticks(self, ws, brokerTicks):
    ticks = []
    for bTick in brokerTicks:
      isd = Instruments.getInstrumentDataByToken(bTick['instrument_token'])
      tradingSymbol = isd['tradingsymbol']
      tick = TickData(tradingSymbol)
      tick.lastTradedPrice = bTick['last_price']
      tick.lastTradedQuantity = bTick['last_quantity']
      tick.avgTradedPrice = bTick['average_price']
      tick.volume = bTick['volume']
      tick.totalBuyQuantity = bTick['buy_quantity']
      tick.totalSellQuantity = bTick['sell_quantity']
      tick.open = bTick['ohlc']['open']
      tick.high = bTick['ohlc']['high']
      tick.low = bTick['ohlc']['low']
      tick.close = bTick['ohlc']['close']
      tick.change = bTick['change']
      ticks.append(tick)
    self.onNewTicks(ticks)
```

ZerodhaTicker is the Zerodha-specific implementation of the BaseTicker contract that wires live market feed, instrument metadata and login state into the framework so TradeManager, strategies and tests can receive normalized ticks. On construction it initializes as a BaseTicker for the Zerodha broker and therefore inherits the brokerLogin handle supplied by Controller. When startTicker is invoked it asks the brokerLogin for the stored broker application details and access token; if the access token is missing it logs an error and aborts startup, otherwise it creates a Kite websocket client using the broker app details and access token, attaches the Zerodha-specific websocket callbacks (connect, close, error, reconnect, no-reconnect, ticks and order-update handlers), records the ticker instance on self and opens the websocket in a non-blocking/threaded mode so the rest of the engine can continue running. stopTicker performs an orderly websocket close. registerSymbols and unregisterSymbols translate framework trading symbols to Zerodha instrument tokens via Instruments.getInstrumentDataBySymbol, log the token lists and then call the underlying Kite subscribe/unsubscribe so the broker will start/stop emitting ticks for those instruments; these methods are the bridge used by TradeManager when a strategy registers its symbols. The on_ticks handler is the normalization step: for each incoming Zerodha tick it looks up instrument metadata by token, constructs a TickData object with the trading symbol (using the TickData constructor you already saw), and populates the TickData fields from the Zerodha payload (last price/quantity, average price, volume, buy/sell queues, OHLC and change). After converting the whole batch it delegates to BaseTicker.onNewTicks which fans those normalized TickData instances out to registered listeners such as TradeManager.tickerListener or test callbacks. In short, ZerodhaTicker performs Zerodha-specific login verification, connection lifecycle management, symbol-token mapping, raw-to-uniform tick transformation, and subscription control so the rest of sdoosa-algo-trade-python-master_cleaned can operate on a consistent tick model regardless of the broker.

```python
# file path: src/loginmgmt/BaseLogin.py
class BaseLogin:
  def __init__(self, brokerAppDetails):
    self.brokerAppDetails = brokerAppDetails
    self.broker = brokerAppDetails.broker
  def login(self, args):
    pass
  def setBrokerHandle(self, brokerHandle):
    self.brokerHandle = brokerHandle
  def setAccessToken(self, accessToken):
    self.accessToken = accessToken
  def getBrokerAppDetails(self):
    return self.brokerAppDetails
  def getAccessToken(self):
    return self.accessToken
  def getBrokerHandle(self):
    return self.brokerHandle
```

BaseLogin provides the centralized, broker-agnostic container for authentication state and the simple contract that broker adapters implement to establish a live session. When an instance is created it keeps the BrokerAppDetails object and the broker name so subclasses have the configured credentials and identifier available. The login method is intentionally empty at this level and is overridden by broker-specific classes such as ZerodhaLogin; Controller.handleBrokerLogin builds the appropriate subclass and calls its login implementation, and that implementation is expected to perform the broker-specific handshake, obtain an API client (the broker handle) and any access token, and then populate BaseLogin by calling setBrokerHandle and setAccessToken. Those setters persist the brokerHandle and accessToken as instance attributes, while getBrokerHandle, get

```python
# file path: src/models/TickData.py
class TickData:
  def __init__(self, tradingSymbol):
    self.tradingSymbol = tradingSymbol
    self.lastTradedPrice = 0
    self.lastTradedQuantity = 0
    self.avgTradedPrice = 0
    self.volume = 0
    self.totalBuyQuantity = 0
    self.totalSellQuantity = 0
    self.open = 0
    self.high = 0
    self.low = 0
    self.close = 0
    self.change = 0
```

TickData is the lightweight, normalized container the Zerodha adapter uses to turn a raw broker tick into a predictable, numeric object that the strategy, risk and execution layers can consume. ZerodhaTicker maps the incoming broker token to a tradingsymbol via Instruments.getInstrumentDataByToken and then constructs a TickData for that symbol; the on_ticks path fills the TickData fields so downstream listeners accessed through BaseTicker and TradeManager receive a consistent shape. The class exposes the canonical per-tick fields you expect for real-time decisioning — last traded price and quantity, average traded price, aggregate volume, total buy and sell quantity, and the OHLC (open, high, low, close) plus change — and it initializes all of them to zero to guarantee defined numeric values during tick delivery. Because it contains only the essentials, TickData is optimized for fast in-memory propagation in the ticker pipeline and for simple checks like price-trigger comparisons, volume filters and short-term movement calculations performed by the Controller/strategy code; it mirrors the same domain-pattern used across the project for small DTOs such as Order and Quote but intentionally omits the extra derivatives present on Quote (for example open interest and circuit limits) to keep the tick object minimal and fast for live processing.

```python
# file path: src/instruments/Instruments.py
import os
import logging
import json
from config.Config import getServerConfig, getTimestampsData, saveTimestampsData
from core.Controller import Controller
from utils.Utils import Utils
```

The Instruments component brings in standard runtime and persistence helpers: os is used for filesystem interactions when reading or writing instrument timestamp and metadata files, logging is used so Instruments can emit the same kind of runtime diagnostics other core pieces produce (recall Controller and TradeManager also log), and json is used to serialize and deserialize the timestamp/metadata payloads. From config.Config it imports getServerConfig, getTimestampsData, and saveTimestampsData so Instruments can read server-level settings and load or persist the cached timestamp/metadata that strategies rely on. Controller from core.Controller is imported so Instruments can participate in the same orchestration surface and access controller-level facilities if needed, and Utils from utils.Utils supplies shared helper routines used during normalization and format conversions. This set of imports follows the project pattern where modules pair filesystem/config access, logging, controller integration, and utility helpers, but Instruments is distinct in that it explicitly imports the timestamp persistence functions rather than broker tickers or order managers found in other modules.

```python
# file path: src/instruments/Instruments.py
        Instruments.saveInstruments(instrumentsList)
    if len(instrumentsList) == 0:
      print("Could not fetch/load instruments data. Hence exiting the app.")
      logging.error("Could not fetch/load instruments data. Hence exiting the app.");
      exit(-2)
    Instruments.symbolToInstrumentMap = {}
    Instruments.tokenToInstrumentMap = {}
    for isd in instrumentsList:
      tradingSymbol = isd['tradingsymbol']
      instrumentToken = isd['instrument_token']
      Instruments.symbolToInstrumentMap[tradingSymbol] = isd
      Instruments.tokenToInstrumentMap[instrumentToken] = isd
    logging.info('Fetching instruments done. Instruments count = %d', len(instrumentsList))
    Instruments.instrumentsList = instrumentsList 
    return instrumentsList
  @staticmethod
  def getInstrumentDataBySymbol(tradingSymbol):
    return Instruments.symbolToInstrumentMap[tradingSymbol]
  @staticmethod
  def getInstrumentDataByToken(instrumentToken):
    return Instruments.tokenToInstrumentMap[instrumentToken]
```

After obtaining a list of instruments, Instruments_part2 persists that list by calling Instruments.saveInstruments so the normalized metadata is available on disk for subsequent runs; if the list is empty it prints an error, logs an error and terminates the process with a non-zero exit code. It then creates two in-memory indices to make lookups efficient for downstream consumers: one that maps trading symbols to the instrument metadata and another that maps instrument tokens to the same metadata objects. The code iterates through the loaded instrument dictionaries, extracts the trading symbol and instrument token for each entry, and populates Instruments.symbolToInstrumentMap and Instruments.tokenToInstrumentMap accordingly. After building these maps it logs the total instruments count, assigns the full list to Instruments.instrumentsList so callers that need the complete array can access it, and returns the list. The two static accessors getInstrumentDataBySymbol and getInstrumentDataByToken simply return the corresponding entry from those maps, providing BNFORB30Min, OptionSelling, ShortStraddleBNF, ZerodhaTicker and TradeManager with fast symbol/token resolution; the persistence call ties back to Instruments_part1 behavior where saveInstruments updates timestamps and writes the instruments file, and the timestamp helpers and Utils functions referenced elsewhere determine when fresh fetches and saves happen.

```python
# file path: src/instruments/Instruments.py
class Instruments:
  instrumentsList = None
  symbolToInstrumentMap = None
  tokenToInstrumentMap = None
  @staticmethod
  def shouldFetchFromServer():
    timestamps = getTimestampsData()
    if 'instrumentsLastSavedAt' not in timestamps:
      return True
    lastSavedTimestamp = timestamps['instrumentsLastSavedAt']
    nowEpoch = Utils.getEpoch()
    if nowEpoch - lastSavedTimestamp >= 24 * 60* 60:
      logging.info("Instruments: shouldFetchFromServer() returning True as its been 24 hours since last fetch.")
      return True
    return False
  @staticmethod
  def updateLastSavedTimestamp():
    timestamps = getTimestampsData()
    timestamps['instrumentsLastSavedAt'] = Utils.getEpoch()
    saveTimestampsData(timestamps)
  @staticmethod
  def loadInstruments():
    serverConfig = getServerConfig()
    instrumentsFilepath = os.path.join(serverConfig['deployDir'], 'instruments.json')
    if os.path.exists(instrumentsFilepath) == False:
      logging.warn('Instruments: instrumentsFilepath %s does not exist', instrumentsFilepath)
      return [] 
    isdFile = open(instrumentsFilepath, 'r')
    instruments = json.loads(isdFile.read())
    logging.info('Instruments: loaded %d instruments from file %s', len(instruments), instrumentsFilepath)
    return instruments
  @staticmethod
  def saveInstruments(instruments = []):
    serverConfig = getServerConfig()
    instrumentsFilepath = os.path.join(serverConfig['deployDir'], 'instruments.json')
    with open(instrumentsFilepath, 'w') as isdFile:
      json.dump(instruments, isdFile, indent=2, default=str)
    logging.info('Instruments: Saved %d instruments to file %s', len(instruments), instrumentsFilepath)
    Instruments.updateLastSavedTimestamp()
  @staticmethod
  def fetchInstrumentsFromServer():
    instrumentsList = []
    try:
      brokerHandle = Controller.getBrokerLogin().getBrokerHandle()
      logging.info('Going to fetch instruments from server...')
      instrumentsList = brokerHandle.instruments('NSE')
      instrumentsListFnO = brokerHandle.instruments('NFO')
      instrumentsList.extend(instrumentsListFnO)
      logging.info('Fetched %d instruments from server.', len(instrumentsList))
    except Exception as e:
      logging.exception("Exception while fetching instruments from server")
    return instrumentsList
  @staticmethod
  def fetchInstruments():
    if Instruments.instrumentsList:
      return Instruments.instrumentsList
    instrumentsList = Instruments.loadInstruments()
    if len(instrumentsList) == 0 or Instruments.shouldFetchFromServer() == True:
      instrumentsList = Instruments.fetchInstrumentsFromServer()
      if len(instrumentsList) > 0:
```

Instruments implements the cache-and-fetch layer that makes normalized instrument metadata available to the rest of the framework so the ticker, order managers and strategies can resolve trading symbols and instrument tokens reliably. It keeps an in-memory cache via instrumentsList, symbolToInstrumentMap and tokenToInstrumentMap and provides the orchestration to load from disk, refresh from the broker and persist metadata and timestamps. The decision to refetch is encapsulated in shouldFetchFromServer, which consults persisted timestamps via getTimestampsData and uses Utils.getEpoch to decide whether the last save is missing or older than 24 hours; that logic prevents unnecessary network calls while ensuring daily refreshes. updateLastSavedTimestamp writes the current epoch into the timestamps payload and delegates persistence to saveTimestampsData so the refresh decision persists across runs. loadInstruments reads the instruments.json file from the deployDir obtained via getServerConfig, logs when the file is absent and returns an empty list in that case. saveInstruments serializes the supplied instrument list back to instruments.json, logs the save, and updates the instrumentsLastSavedAt timestamp immediately after writing. fetchInstrumentsFromServer obtains the active broker handle through Controller.getBrokerLogin (which ties back to BaseLogin) and asks the broker for NSE and NFO instrument lists, combining them and catching/logging any exceptions that occur during the network call. fetchInstruments is the top-level accessor: it returns the in-memory cache if populated, otherwise it tries loadInstruments, and when the on-disk list is empty or shouldFetchFromServer indicates a refresh, it calls fetchInstrumentsFromServer; the rest of the post-fetch work (persisting with saveInstruments, building symbol and token maps and failing fast if nothing is available) is performed by Instruments_part2. Overall, Instruments_part1 coordinates when and how instrument metadata is read, refreshed and timestamped so the rest of the system (ZerodhaTicker, order managers and strategies) can rely on stable symbol-to-token mappings and avoid unnecessary broker queries.

```python
# file path: src/trademgmt/TradeManager.py
    logging.info('TradeManager: Successfully loaded %d trades from json file %s', len(TradeManager.trades), tradesFilepath)
  @staticmethod
  def saveAllTradesToFile():
    tradesFilepath = os.path.join(TradeManager.intradayTradesDir, 'trades.json')
    with open(tradesFilepath, 'w') as tFile:
      json.dump(TradeManager.trades, tFile, indent=2, cls=TradeEncoder)
    logging.info('TradeManager: Saved %d trades to file %s', len(TradeManager.trades), tradesFilepath)
  @staticmethod
  def addNewTrade(trade):
    if trade == None:
      return
    logging.info('TradeManager: addNewTrade called for %s', trade)
    for tr in TradeManager.trades:
      if tr.equals(trade):
        logging.warn('TradeManager: Trade already exists so not adding again. %s', trade)
        return
    TradeManager.trades.append(trade)
    logging.info('TradeManager: trade %s added successfully to the list', trade.tradeID)
    if trade.tradingSymbol not in TradeManager.registeredSymbols:
      TradeManager.ticker.registerSymbols([trade.tradingSymbol])
      TradeManager.registeredSymbols.append(trade.tradingSymbol)
    strategyInstance = TradeManager.strategyToInstanceMap[trade.strategy]
    if strategyInstance != None:
      strategyInstance.addTradeToList(trade)
  @staticmethod
  def disableTrade(trade, reason):
    if trade != None:
      logging.info('TradeManager: Going to disable trade ID %s with the reason %s', trade.tradeID, reason)
      trade.tradeState = TradeState.DISABLED
  @staticmethod
  def tickerListener(tick):
    TradeManager.symbolToCMPMap[tick.tradingSymbol] = tick.lastTradedPrice 
    for strategy in TradeManager.strategyToInstanceMap:
      longTrade = TradeManager.getUntriggeredTrade(tick.tradingSymbol, strategy, Direction.LONG)
      shortTrade = TradeManager.getUntriggeredTrade(tick.tradingSymbol, strategy, Direction.SHORT)
      if longTrade == None and shortTrade == None:
        continue
      strategyInstance = TradeManager.strategyToInstanceMap[strategy]
      if longTrade != None:
        if strategyInstance.shouldPlaceTrade(longTrade, tick):
          isSuccess = TradeManager.executeTrade(longTrade)
          if isSuccess == True:
            longTrade.tradeState = TradeState.ACTIVE
            longTrade.startTimestamp = Utils.getEpoch()
            continue
      if shortTrade != None:
        if strategyInstance.shouldPlaceTrade(shortTrade, tick):
          isSuccess = TradeManager.executeTrade(shortTrade)
          if isSuccess == True:
            shortTrade.tradeState = TradeState.ACTIVE
            shortTrade.startTimestamp = Utils.getEpoch()
  @staticmethod
  def getUntriggeredTrade(tradingSymbol, strategy, direction):
    trade = None
    for tr in TradeManager.trades:
      if tr.tradeState == TradeState.DISABLED:
        continue
      if tr.tradeState != TradeState.CREATED:
        continue
      if tr.tradingSymbol != tradingSymbol:
```

The code continues the TradeManager role of gluing strategy signals to execution and bookkeeping by persisting and managing the in-memory trade list, wiring market data to strategy decision logic, and triggering execution when strategies indicate a trade should be placed. saveAllTradesToFile serializes the in-memory trades list to a trades.json file under the intradayTradesDir using the TradeEncoder and logs the outcome so the day's state is durable. addNewTrade first guards against a null argument, logs the incoming trade, and prevents duplicates by comparing with existing trades using Trade.equals; on successful addition it appends the trade to the shared trades list, logs the new tradeID, ensures the ticker is subscribed to the trade's tradingSymbol (registering the symbol with the ZerodhaTicker and recording it in registeredSymbols) and, if a strategy instance exists for the trade's strategy name, hands the trade to that strategy via addTradeToList so the strategy can track it. disableTrade simply marks a trade as disabled and records a logged reason. tickerListener receives normalized TickData (recall ZerodhaTicker provides ticks), updates symbolToCMPMap with the latest price, and then for every registered strategy it queries for any untriggered long and short trades for the tick's symbol using getUntriggeredTrade; if a trade exists it asks the corresponding strategy instance whether it should place the trade via shouldPlaceTrade, and when the strategy says yes it calls TradeManager.executeTrade (which invokes the order manager to place orders) and, on success, moves the trade into the ACTIVE state and stamps its startTimestamp using Utils.getEpoch. getUntriggeredTrade scans the central trades list, skipping disabled trades and anything not in the CREATED state, and returns the first trade that matches the given tradingSymbol, strategy and direction, which is how the listener identifies candidate trades that are waiting for market conditions to execute.

```python
# file path: src/trademgmt/Trade.py
import logging
from trademgmt.TradeState import TradeState
from models.ProductType import ProductType
from utils.Utils import Utils
```

The file brings in a small, focused set of runtime dependencies that let the Trade model represent and log its lifecycle. logging provides the same diagnostic channel other core modules use so Trade can emit runtime messages consistent with Controller, TradeManager and Instruments. TradeState supplies the canonical lifecycle enumeration that Trade uses to track and transition an individual trade through states like NEW, OPEN, CLOSED, etc., so the trade-management layer has a single source of truth for status logic. ProductType supplies the contract-level classification (for example intra-day vs delivery products) that Trade uses to compute sizing, margin and exit behavior according to broker/product rules. Utils provides the shared helper functions used across the project (timestamping, numeric rounding, simple serialization and other small utilities) so Trade can reuse common operations instead of reimplementing them. This import pattern mirrors other modules that commonly import logging, ProductType and Utils; the difference here is that Trade imports TradeState specifically because it models lifecycle transitions, whereas other files often pair ProductType and Utils with execution, order or strategy-specific models like Direction or OrderManager.

```python
# file path: src/trademgmt/Trade.py
class Trade:
  def __init__(self, tradingSymbol = None):
    self.exchange = "NSE" 
    self.tradeID = Utils.generateTradeID() 
    self.tradingSymbol = tradingSymbol
    self.strategy = ""
    self.direction = ""
    self.productType = ProductType.MIS
    self.isFutures = False 
    self.isOptions = False 
    self.optionType = None 
    self.placeMarketOrder = False 
    self.intradaySquareOffTimestamp = None 
    self.requestedEntry = 0 
    self.entry = 0 
    self.qty = 0 
    self.filledQty = 0 
    self.initialStopLoss = 0 
    self.stopLoss = 0 
    self.target = 0 
    self.cmp = 0 
    self.tradeState = TradeState.CREATED 
    self.timestamp = None 
    self.createTimestamp = Utils.getEpoch() 
    self.startTimestamp = None 
    self.endTimestamp = None 
    self.pnl = 0 
    self.pnlPercentage = 0 
    self.exit = 0 
    self.exitReason = None 
    self.entryOrder = None 
    self.slOrder = None 
    self.targetOrder = None 
  def equals(self, trade): 
    if trade == None:
      return False
    if self.tradeID == trade.tradeID:
      return True
    if self.tradingSymbol != trade.tradingSymbol:
      return False
    if self.strategy != trade.strategy:
      return False  
    if self.direction != trade.direction:
      return False
    if self.productType != trade.productType:
      return False
    if self.requestedEntry != trade.requestedEntry:
      return False
    if self.qty != trade.qty:
      return False
    if self.timestamp != trade.timestamp:
      return False
    return True
  def __str__(self):
    return "ID=" + str(self.tradeID) + ", state=" + self.tradeState + ", symbol=" + self.tradingSymbol \
      + ", strategy=" + self.strategy + ", direction=" + self.direction \
      + ", productType=" + self.productType + ", reqEntry=" + str(self.requestedEntry) \
      + ", stopLoss=" + str(self.stopLoss) + ", target=" + str(self.target) \
      + ", entry=" + str(self.entry) + ", exit=" + str(self.exit) \
      + ", profitLoss" + str(self.pnl)
```

Trade is the domain object the trade-management layer uses to represent one executable position and track its lifecycle, risk parameters and P&L as strategies and the manager move it from creation to completion. On construction a Trade is given a normalized trading symbol, a unique tradeID produced by Utils.generateTradeID and a createTimestamp from Utils.getEpoch, and sensible defaults such as exchange set to NSE and productType set to ProductType.MIS; those initial fields let strategy code like BNFORB30Min, OptionSelling, SampleStrategy and ShortStraddleBNF instantiate trades with minimal boilerplate. The instance fields capture the full state machine and data needed by TradeManager and OrderManager: requestedEntry (the price the strategy requested), entry (the executed price), qty and filledQty (order lifecycle vs fills), initialStopLoss and stopLoss (to support trailing SL logic obtained from BaseStrategy.getTrailingSL), target, cmp for the latest market price (updated from ZerodhaTicker.on_ticks via TradeManager.tickerListener), tradeState (starts as TradeState.CREATED), timestamps for start/end and an optional intradaySquareOffTimestamp used by TradeManager.trackAndUpdateAllTrades to force exits. Flags such as isFutures, isOptions and optionType plus interactions with Utils.prepareMonthlyExpiryFuturesSymbol, Utils.getTodayDateStr and Utils.isTodayWeeklyExpiryDay let strategy code and symbol helpers produce correct FnO instrument names when a trade is for futures or options. Placeholders for entryOrder, slOrder and targetOrder link the trade to Order objects managed by the broker adaptor and used by TradeManager.executeTrade and fetch/update flows; TradeManager.convertJSONToTrade maps persisted JSON back into Trade instances so trades survive restarts and saveAllTradesToFile persists them. The equals method implements a practical deduplication/identity check: it first accepts matching tradeID, otherwise it verifies key attributes (symbol, strategy, direction, productType, requestedEntry, qty and timestamp) to decide if two Trade instances represent the same logical trade, which is how higher-level flows avoid duplicate placement. The string method produces a compact human-readable summary used in logging. Conceptually Trade follows the same simple DTO pattern as Order, but while Order contains broker-order-level metadata and status, Trade aggregates lifecycle, risk controls, timestamps and P&L so strategies, the OrderManager and TradeManager can coordinate execution, trailing SL updates, square-offs and final exit accounting.

```python
# file path: src/strategies/BaseStrategy.py
      self.process()
      now = datetime.now()
      waitSeconds = 30 - (now.second % 30) 
      time.sleep(waitSeconds)
  def shouldPlaceTrade(self, trade, tick):
    if trade == None:
      return False
    if trade.qty == 0:
      TradeManager.disableTrade(trade, 'InvalidQuantity')
      return False
    now = datetime.now()
    if now > self.stopTimestamp:
      TradeManager.disableTrade(trade, 'NoNewTradesCutOffTimeReached')
      return False
    numOfTradesPlaced = TradeManager.getNumberOfTradesPlacedByStrategy(self.getName())
    if numOfTradesPlaced >= self.maxTradesPerDay:
      TradeManager.disableTrade(trade, 'MaxTradesPerDayReached')
      return False
    return True
  def addTradeToList(self, trade):
    if trade != None:
      self.trades.append(trade)
  def getQuote(self, tradingSymbol):
    return Quotes.getQuote(tradingSymbol, self.isFnO)
  def getTrailingSL(self, trade):
    return 0
```

Within the BaseStrategy lifecycle, after a concrete strategy's process method runs the run loop enforces a half‑minute cadence by computing the seconds remaining to the next 30‑second boundary and sleeping that amount so strategies are invoked on a predictable tick rhythm. The shouldPlaceTrade method is the standard pre‑trade gate used by all derived strategies: it returns false for a missing trade, disables and rejects any trade with zero quantity by delegating to TradeManager.disableTrade (which persists the disabled state), checks the strategy's stopTimestamp (set by subclasses such as BNFORB30Min and OptionSelling) and disables new trades when the cut‑off time is reached, and consults TradeManager.getNumberOfTradesPlacedByStrategy to enforce maxTradesPerDay, disabling the trade if the daily limit is hit; only when all these checks pass does it allow placement. addTradeToList is a small helper that appends a non‑null Trade into the strategy's local trades collection that was populated from TradeManager.getAllTradesByStrategy during initialization. getQuote centralizes quote access for strategies by delegating to Quotes.getQuote with the strategy's isFnO flag so the same call returns normalized market data whether the symbol is cash or derivatives. getTrailingSL is a simple overridable hook that returns zero by default; concrete strategies (for example ShortStraddleBNF) override it to provide dynamic trailing stop logic that TradeManager.trackAndUpdateAllTrades can use. Together these methods provide the standardized decision points and utilities that wire a concrete strategy into TradeManager, Quotes and the runtime timing expectations of the algo framework.

```python
# file path: src/core/Quotes.py
import logging
from core.Controller import Controller
from models.Quote import Quote
```

The file pulls in three things it needs to perform the Quotes role: a logging facility so it can emit the same runtime diagnostics other core pieces produce (Controller and TradeManager also log), the Controller so Quotes can hook into the framework’s centralized runtime services (connection/authentication state, event dispatch and feed wiring) and register handlers for incoming market data, and the Quote model so incoming raw ticks can be normalized into the predictable quote objects that strategies and tests will consume. This mirrors a common pattern you’ve seen elsewhere where modules import logging and Controller to participate in the shared runtime; unlike some files that also bring in configuration or broker-specific login adaptors, Quotes keeps its imports minimal—just the controller for runtime integration and the Quote class for shaping outbound data—because its responsibility is focused on feed handling and normalization rather than configuration or login orchestration.

```python
# file path: src/core/Quotes.py
class Quotes:
  @staticmethod
  def getQuote(tradingSymbol, isFnO = False):
    broker = Controller.getBrokerName()
    brokerHandle = Controller.getBrokerLogin().getBrokerHandle()
    quote = None
    if broker == "zerodha":
      key = ('NFO:' + tradingSymbol) if isFnO == True else ('NSE:' + tradingSymbol)
      bQuoteResp = brokerHandle.quote(key) 
      bQuote = bQuoteResp[key]
      quote = Quote(tradingSymbol)
      quote.tradingSymbol = tradingSymbol
      quote.lastTradedPrice = bQuote['last_price']
      quote.lastTradedQuantity = bQuote['last_quantity']
      quote.avgTradedPrice = bQuote['average_price']
      quote.volume = bQuote['volume']
      quote.totalBuyQuantity = bQuote['buy_quantity']
      quote.totalSellQuantity = bQuote['sell_quantity']
      ohlc = bQuote['ohlc']
      quote.open = ohlc['open']
      quote.high = ohlc['high']
      quote.low = ohlc['low']
      quote.close = ohlc['close']
      quote.change = bQuote['net_change']
      quote.oiDayHigh = bQuote['oi_day_high']
      quote.oiDayLow = bQuote['oi_day_low']
      quote.lowerCiruitLimit = bQuote['lower_circuit_limit']
      quote.upperCircuitLimit = bQuote['upper_circuit_limit']
    else:
      quote = None
    return quote
  @staticmethod
  def getCMP(tradingSymbol):
    quote = Quotes.getQuote(tradingSymbol)
    if quote:
      return quote.lastTradedPrice
    else:
      return 0
```

Quotes provides the mid-level adapter that sits between the Controller-managed broker session and the rest of the framework (tests and strategies) by turning a broker-specific quote snapshot into the framework's normalized Quote object. Its static getQuote method first asks Controller for the active broker name and then fetches the authenticated broker handle from Controller.getBrokerLogin, relying on the BaseLogin-managed brokerHandle that was created during login. For the Zerodha path it composes the exchange-prefixed instrument key depending on the isFnO flag, calls the broker handle's quote method to retrieve the raw broker payload, and then constructs a Quote instance and copies the relevant numeric fields from the broker response into the Quote: last traded price/quantity, average price, volume, buy/sell aggregate quantities, the nested OHLC values, net change, open interest day high/low and circuit limits. If the active broker is not recognized the method returns None. The static getCMP method is a thin convenience wrapper that returns lastTradedPrice from getQuote or zero when no quote is available; strategies and helpers (for example BaseStrategy.getTrailingSL and Test.testMisc) call getCMP/getQuote to obtain a consistent numeric current market price or a normalized Quote object without needing to know Zerodha's payload shape.

```python
# file path: src/models/Quote.py
class Quote:
  def __init__(self, tradingSymbol):
    self.tradingSymbol = tradingSymbol
    self.lastTradedPrice = 0
    self.lastTradedQuantity = 0
    self.avgTradedPrice = 0
    self.volume = 0
    self.totalBuyQuantity = 0
    self.totalSellQuantity = 0
    self.open = 0
    self.high = 0
    self.low = 0
    self.close = 0
    self.change = 0
    self.oiDayHigh = 0
    self.oiDayLow = 0
    self.lowerCiruitLimit = 0
    self.upperCircuitLimit = 0
```

Quote is the lightweight, broker-agnostic data container that represents a single instrument’s market snapshot for the rest of the engine to consume. Its constructor records the provided tradingSymbol and initializes a fixed set of numeric attributes (for example lastTradedPrice, lastTradedQuantity, avgTradedPrice, volume, totalBuyQuantity, totalSellQuantity, open, high, low, close, change, oiDayHigh, oiDayLow, lowerCiruitLimit and upperCircuitLimit) to zero so downstream code can rely on predictable, numeric values even when a broker returns partial data. In the runtime flow, Quotes.getQuote obtains the broker name from Controller and the broker handle from BaseLogin (recall ZerodhaTicker and BaseLogin provide the live feed and login contract), fetches the raw broker quote and maps broker fields into a Quote instance; this mapping is what normalizes broker-specific payloads into the uniform Quote shape. Because Quote mirrors most of TickData’s price/size fields but also carries open-interest and circuit limit fields, it serves as the richer snapshot that components such as BaseStrategy.getTrailingSL, the order manager, the Quotes aggregator and Test.tickerListener use directly; helper methods like Quotes.getCMP then simply read lastTradedPrice from a Quote (or fall back to zero) to supply a current market price.

```python
# file path: src/trademgmt/TradeManager.py
  def cancelTargetOrder(trade):
    if trade.targetOrder == None:
      return
    if trade.targetOrder.orderStatus == OrderStatus.CANCELLED:
      return
    try:
      TradeManager.getOrderManager().cancelOrder(trade.targetOrder)
    except Exception as e:
      logging.error('TradeManager: Failed to cancel Target order %s for tradeID %s: Error => %s', trade.targetOrder.orderId, trade.tradeID, str(e))
    logging.info('TradeManager: Successfully cancelled Target order %s for tradeID %s', trade.targetOrder.orderId, trade.tradeID)
  @staticmethod
  def setTradeToCompleted(trade, exit, exitReason = None):
    trade.tradeState = TradeState.COMPLETED
    trade.exit = exit
    trade.exitReason = exitReason if trade.exitReason == None else trade.exitReason
    trade.endTimestamp = Utils.getEpoch()
    trade = Utils.calculateTradePnl(trade)
    logging.info('TradeManager: setTradeToCompleted strategy = %s, symbol = %s, qty = %d, entry = %f, exit = %f, pnl = %f, exit reason = %s', trade.strategy, trade.tradingSymbol, trade.filledQty, trade.entry, trade.exit, trade.pnl, trade.exitReason)
  @staticmethod
  def squareOffTrade(trade, reason = TradeExitReason.SQUARE_OFF):
    logging.info('TradeManager: squareOffTrade called for tradeID %s with reason %s', trade.tradeID, reason)
    if trade == None or trade.tradeState != TradeState.ACTIVE:
      return
    trade.exitReason = reason
    if trade.entryOrder != None:
      if trade.entryOrder.orderStatus == OrderStatus.OPEN:
        TradeManager.cancelEntryOrder(trade)
    if trade.slOrder != None:
      TradeManager.cancelSLOrder(trade)
    if trade.targetOrder != None:
      logging.info('TradeManager: changing target order %s to MARKET to exit position for tradeID %s', trade.targetOrder.orderId, trade.tradeID)
      TradeManager.getOrderManager().modifyOrderToMarket(trade.targetOrder)
    else:
      logging.info('TradeManager: placing new target order to exit position for tradeID %s', trade.tradeID)
      TradeManager.placeTargetOrder(trade, True)
  @staticmethod
  def getOrderManager():
    orderManager = None
    brokerName = Controller.getBrokerName()
    if brokerName == "zerodha":
      orderManager = ZerodhaOrderManager()
    return orderManager
  @staticmethod
  def getNumberOfTradesPlacedByStrategy(strategy):
    count = 0
    for trade in TradeManager.trades:
      if trade.strategy != strategy:
        continue
      if trade.tradeState == TradeState.CREATED or trade.tradeState == TradeState.DISABLED:
        continue
      count += 1
    return count
  @staticmethod
  def getAllTradesByStrategy(strategy):
    tradesByStrategy = []
    for trade in TradeManager.trades:
      if trade.strategy == strategy:
        tradesByStrategy.append(trade)
    return tradesByStrategy
  @staticmethod
```

TradeManager_part6 continues the lifecycle orchestration responsibilities by implementing helper actions used when exiting or querying trades and by selecting the concrete order manager for execution. cancelTargetOrder ensures any outstanding target order for a trade is only acted on if it exists and is not already cancelled, then delegates the cancellation to the order manager and logs errors or successful cancellation; it relies on the same order manager abstraction that TradeManager_part3 used to place entry orders and that ZerodhaOrderManager implements. setTradeToCompleted centralizes the finalization of a trade: it flips the trade state to COMPLETED, records the numeric exit and preserves any previously-set exitReason if present, stamps the end time using Utils.getEpoch, recomputes PnL via Utils.calculateTradePnl, and emits a summary log so bookkeeping and downstream reporting have a single canonical place to mark trade termination. squareOffTrade implements an orderly intraday or forced exit path: it first validates the trade is active, records the exit reason, cancels any still-open entry order and cancels the stop-loss using the cancelSLOrder logic covered in TradeManager_part5, and then either converts an existing target order into a market exit via the broker adaptor’s modify-to-market facility or places a fresh market target order via placeTargetOrder (also in TradeManager_part5); this is how the framework enforces square-off and emergency exits across strategies. getOrderManager is the factory that maps the configured broker name from Controller into a concrete order manager instance; in the current configuration it returns a ZerodhaOrderManager for the "zerodha" broker so all higher-level trade actions route to the broker-specific implementation. getNumberOfTradesPlacedByStrategy and getAllTradesByStrategy are simple query utilities over the TradeManager.trades collection that respectively count active/finished trades (excluding CREATED or DISABLED states) for a given strategy and return the full list of trades owned by a strategy; these are the helper functions strategies and throttle logic call to enforce per-strategy limits and to inspect past/ongoing positions. Overall, these methods glue strategy decisions to broker actions, consolidate trade completion bookkeeping, and provide the adapter selection and simple queries that other TradeManager parts and strategy code rely on.

```python
# file path: src/ordermgmt/ZerodhaOrderManager.py
import logging
from ordermgmt.BaseOrderManager import BaseOrderManager
from ordermgmt.Order import Order
from models.ProductType import ProductType
from models.OrderType import OrderType
from models.Direction import Direction
from models.OrderStatus import OrderStatus
from utils.Utils import Utils
```

The import list wires in the small set of building blocks ZerodhaOrderManager needs to act as the Zerodha-specific implementation of the order-management layer. logging provides the runtime diagnostics that let the order manager emit the same standardized runtime traces other core pieces produce so TradeManager and tests can correlate events. BaseOrderManager is the broker-agnostic abstract contract that ZerodhaOrderManager implements to translate generic order actions into platform calls and lifecycle events. Order is the normalized in-memory order representation the manager will create, update and return as it places, modifies and cancels orders. ProductType, OrderType, Direction and OrderStatus are the domain model enums that let the manager map framework-level semantics (market vs limit, intraday vs delivery, buy vs sell, live order state) to Zerodha API fields and to track order lifecycle transitions. Utils supplies small runtime helpers used by the manager for common tasks such as parameter normalization, numeric rounding and timestamp/ID handling so the class can keep its logic concise. These imports follow the same pattern seen in other modules that implement broker adapters or orchestration: everyone brings in logging, the core Order model and the shared model enums plus the common Utils helpers so different components speak the same domain language when TradeManager, tests or other subsystems interact with ZerodhaOrderManager.

```python
# file path: src/ordermgmt/ZerodhaOrderManager.py
class ZerodhaOrderManager(BaseOrderManager):
  def __init__(self):
    super().__init__("zerodha")
  def placeOrder(self, orderInputParams):
    logging.info('%s: Going to place order with params %s', self.broker, orderInputParams)
    kite = self.brokerHandle
    try:
      orderId = kite.place_order(
        variety=kite.VARIETY_REGULAR,
        exchange=kite.EXCHANGE_NFO if orderInputParams.isFnO == True else kite.EXCHANGE_NSE,
        tradingsymbol=orderInputParams.tradingSymbol,
        transaction_type=self.convertToBrokerDirection(orderInputParams.direction),
        quantity=orderInputParams.qty,
        price=orderInputParams.price,
        trigger_price=orderInputParams.triggerPrice,
        product=self.convertToBrokerProductType(orderInputParams.productType),
        order_type=self.convertToBrokerOrderType(orderInputParams.orderType))
      logging.info('%s: Order placed successfully, orderId = %s', self.broker, orderId)
      order = Order(orderInputParams)
      order.orderId = orderId
      order.orderPlaceTimestamp = Utils.getEpoch()
      order.lastOrderUpdateTimestamp = Utils.getEpoch()
      return order
    except Exception as e:
      logging.info('%s Order placement failed: %s', self.broker, str(e))
      raise Exception(str(e))
  def modifyOrder(self, order, orderModifyParams):
    logging.info('%s: Going to modify order with params %s', self.broker, orderModifyParams)
    kite = self.brokerHandle
    try:
      orderId = kite.modify_order(
        variety=kite.VARIETY_REGULAR,
        order_id=order.orderId,
        quantity=orderModifyParams.newQty if orderModifyParams.newQty > 0 else None,
        price=orderModifyParams.newPrice if orderModifyParams.newPrice > 0 else None,
        trigger_price=orderModifyParams.newTriggerPrice if orderModifyParams.newTriggerPrice > 0 else None,
        order_type=orderModifyParams.newOrderType if orderModifyParams.newOrderType != None else None)
      logging.info('%s Order modified successfully for orderId = %s', self.broker, orderId)
      order.lastOrderUpdateTimestamp = Utils.getEpoch()
      return order
    except Exception as e:
      logging.info('%s Order modify failed: %s', self.broker, str(e))
      raise Exception(str(e))
  def modifyOrderToMarket(self, order):
    logging.info('%s: Going to modify order with params %s', self.broker)
    kite = self.brokerHandle
    try:
      orderId = kite.modify_order(
        variety=kite.VARIETY_REGULAR,
        order_id=order.orderId,
        order_type=kite.ORDER_TYPE_MARKET)
      logging.info('%s Order modified successfully to MARKET for orderId = %s', self.broker, orderId)
      order.lastOrderUpdateTimestamp = Utils.getEpoch()
      return order
    except Exception as e:
      logging.info('%s Order modify to market failed: %s', self.broker, str(e))
      raise Exception(str(e))
  def cancelOrder(self, order):
    logging.info('%s Going to cancel order %s', self.broker, order.orderId)
    kite = self.brokerHandle
```

ZerodhaOrderManager is the Zerodha-specific implementation of the order-management layer that subclasses BaseOrderManager to translate framework order intents into Kite/ Zerodha API calls and to maintain simple lifecycle metadata for orders. Its constructor calls the BaseOrderManager initializer with the broker name so broker and brokerHandle are wired from Controller-managed login state; that brokerHandle is the Kite client used for all subsequent API calls. When placeOrder is called it logs the intent, reads the Kite client from brokerHandle, maps engine-level fields from OrderInputParams into the broker API parameters (choosing exchange based on the isFnO flag, converting direction, product type and order type via the Zerodha-specific converters implemented in other parts, and passing quantity, price and trigger price), invokes the broker place-order call, and on success wraps the response into an Order model instance, stamping orderId, orderPlaceTimestamp and lastOrderUpdateTimestamp using Utils.getEpoch before returning it; on failure it logs and re-raises the exception. modifyOrder follows the same pattern for amendments: it logs, invokes the broker modify-order API and selectively sends only the changed fields (omitting quantity/price/trigger if the new values are non-positive or None), updates the order’s lastOrderUpdateTimestamp and returns the updated Order object, or logs and re-raises on error. modifyOrderToMarket is a focused modifier that changes an existing order to a market order by calling the broker modify API with the market order type, updates the timestamp and returns the order, or logs and re-raises on failure. The class consistently uses the shared utility and model pieces—Order for the normalized order container and Utils.getEpoch for timestamps—and delegates broker-specific value mapping to the convertToBrokerDirection/convertToBrokerProductType/convertToBrokerOrderType helpers defined elsewhere in the ZerodhaOrderManager parts. Overall, ZerodhaOrderManager_part1 implements the synchronous place/modify-to-limit/modify-to-market pathways and error handling used by TradeManager and tests, following the same logging-and-wrap pattern seen in the related cancel and order-book sync logic in the other ZerodhaOrderManager parts.

```python
# file path: src/ordermgmt/Order.py
class Order:
  def __init__(self, orderInputParams = None):
    self.tradingSymbol = orderInputParams.tradingSymbol if orderInputParams != None else ""
    self.exchange = orderInputParams.exchange if orderInputParams != None else "NSE"
    self.productType = orderInputParams.productType if orderInputParams != None else ""
    self.orderType = orderInputParams.orderType if orderInputParams != None else "" 
    self.price = orderInputParams.price if orderInputParams != None else 0
    self.triggerPrice = orderInputParams.triggerPrice if orderInputParams != None else 0 
    self.qty = orderInputParams.qty if orderInputParams != None else 0
    self.orderId = None 
    self.orderStatus = None 
    self.averagePrice = 0 
    self.filledQty = 0 
    self.pendingQty = 0 
    self.orderPlaceTimestamp = None 
    self.lastOrderUpdateTimestamp = None 
    self.message = None 
  def __str__(self):
    return "orderId=" + str(self.orderId) + ", orderStatus=" + str(self.orderStatus) \
      + ", symbol=" + str(self.tradingSymbol) + ", productType=" + str(self.productType) \
      + ", orderType=" + str(self.orderType) + ", price=" + str(self.price) \
      + ", triggerPrice=" + str(self.triggerPrice) + ", qty=" + str(self.qty) \
      + ", filledQty=" + str(self.filledQty) + ", pendingQty=" + str(self.pendingQty) \
      + ", averagePrice=" + str(self.averagePrice)
```

Order defines the canonical order model that the order management and trade lifecycle layers use to represent a broker-level request and its evolving execution state; Test, ZerodhaOrderManager_part1 and TradeManager_part7 all create, inspect and update instances of Order so the rest of the engine has a consistent, simple object to work with. When an Order is constructed it copies intent-level fields from an OrderInputParams instance when provided — tradingSymbol, exchange (defaults to NSE), productType, orderType, price, triggerPrice and qty — and initializes broker-facing, mutable state to neutral values: orderId and orderStatus start unset, numeric execution metrics like averagePrice, filledQty and pendingQty start at zero, and timestamps and message fields start empty. ZerodhaOrderManager_part1 populates the instance after placement (it assigns orderId and sets orderPlaceTimestamp and lastOrderUpdateTimestamp), and runtime callbacks coming through ZerodhaTicker_part1 and BaseTicker.onOrderUpdate update orderStatus, filledQty, pendingQty, averagePrice, lastOrderUpdateTimestamp and message as broker events arrive. TradeManager routines read and sometimes reconstruct Order objects (for example via convertJSONToOrder) to reconcile trade-level state with broker

```python
# file path: src/ordermgmt/BaseOrderManager.py
from core.Controller import Controller
```

The import of Controller brings the framework’s central runtime coordinator into BaseOrderManager so the order manager can participate in the same shared session, event and dispatch infrastructure that other mid-level adapters use. As we saw when examining Quotes, Controller is the single place that owns connection/authentication state, event routing and feed wiring; BaseOrderManager imports Controller for the same reasons — to query the broker session and auth state, register and receive lifecycle events (order updates, fills, connection changes), and route order requests through the controller’s centralized plumbing rather than trying to manage network/session details itself. This follows the common pattern in the codebase where components that implement broker-facing logic import core.Controller to avoid duplicating runtime orchestration; files that additionally need data shapes also import model classes (for example Quote, Segment or ProductType) alongside Controller.

```python
# file path: src/ordermgmt/BaseOrderManager.py
class BaseOrderManager:
  def __init__(self, broker):
    self.broker = broker
    self.brokerHandle = Controller.getBrokerLogin().getBrokerHandle()

  def placeOrder(self, orderInputParams):
    pass

  def modifyOrder(self, order, orderModifyParams):
    pass

  def modifyOrderToMarket(self, order):
    pass

  def cancelOrder(self, order):
    pass

  def fetchAndUpdateAllOrderDetails(self, orders):
    pass

  def convertToBrokerProductType(self, productType):
    return productType

  def convertToBrokerOrderType(self, orderType):
    return orderType

  def convertToBrokerDirection(self, direction):
    return direction
```

BaseOrderManager is the shared scaffold that sits between the Controller-managed broker session and every broker-specific order manager, so it centralizes the common wiring and defines the interface the rest of the engine uses to perform order lifecycle actions. In its initializer it stores the broker identifier and pulls the active broker API client from Controller.getBrokerLogin().getBrokerHandle (recall BaseLogin.getBrokerHandle we examined earlier), making that authenticated client available to subclasses through the brokerHandle attribute. The class then declares the core lifecycle methods—placeOrder, modifyOrder, modifyOrderToMarket, cancelOrder and fetchAndUpdateAllOrderDetails—as no-op stubs that concrete managers must implement; the TradeManager and other runtime pieces call these methods to execute entry orders, update SL/target orders, cancel orders and reconcile local Order objects with the broker order book. BaseOrderManager also provides three conversion helpers that currently return their inputs; these are the extension points that allow the framework to remain broker-agnostic by mapping ProductType, OrderType and Direction into broker-specific constants, and you can see ZerodhaOrderManager overriding those conversions to map to kite constants. Conceptually, when a strategy requests an execution, TradeManager gets the concrete order manager (which inherits from BaseOrderManager), calls placeOrder, the subclass uses brokerHandle to talk to the broker SDK and returns an Order instance (Order fields such as orderId, qty, filledQty and timestamps are populated by the concrete implementation), and later fetchAndUpdateAllOrderDetails is used to pull the broker order book and reconcile order state back into the engine. The class therefore exists to centralize the authenticated handle and the canonical order-management API so different broker adaptors like ZerodhaOrderManager can plug in the broker-specific behavior while the rest of the system (TradeManager, Ticker hooks, HoldingsAPI/PositionsAPI patterns) interacts with a consistent interface.

```python
# file path: src/ordermgmt/ZerodhaOrderManager.py
    try:
      orderId = kite.cancel_order(
        variety=kite.VARIETY_REGULAR,
        order_id=order.orderId)
      logging.info('%s Order cancelled successfully, orderId = %s', self.broker, orderId)
      order.lastOrderUpdateTimestamp = Utils.getEpoch()
      return order
    except Exception as e:
      logging.info('%s Order cancel failed: %s', self.broker, str(e))
      raise Exception(str(e))
  def fetchAndUpdateAllOrderDetails(self, orders):
    logging.info('%s Going to fetch order book', self.broker)
    kite = self.brokerHandle
    orderBook = None
    try:
      orderBook = kite.orders()
    except Exception as e:
      logging.error('%s Failed to fetch order book', self.broker)
      return
    logging.info('%s Order book length = %d', self.broker, len(orderBook))
    numOrdersUpdated = 0
    for bOrder in orderBook:
      foundOrder = None
      for order in orders:
        if order.orderId == bOrder['order_id']:
          foundOrder = order
          break
      if foundOrder != None:
        logging.info('Found order for orderId %s', foundOrder.orderId)
        foundOrder.qty = bOrder['quantity']
        foundOrder.filledQty = bOrder['filled_quantity']
        foundOrder.pendingQty = bOrder['pending_quantity']
        foundOrder.orderStatus = bOrder['status']
        if foundOrder.orderStatus == OrderStatus.CANCELLED and foundOrder.filledQty > 0:
          foundOrder.orderStatus = OrderStatus.COMPLETED
        foundOrder.price = bOrder['price']
        foundOrder.triggerPrice = bOrder['trigger_price']
        foundOrder.averagePrice = bOrder['average_price']
        logging.info('%s Updated order %s', self.broker, foundOrder)
        numOrdersUpdated += 1
    logging.info('%s: %d orders updated with broker order details', self.broker, numOrdersUpdated)
  def convertToBrokerProductType(self, productType):
    kite = self.brokerHandle
    if productType == ProductType.MIS:
      return kite.PRODUCT_MIS
    elif productType == ProductType.NRML:
      return kite.PRODUCT_NRML
    elif productType == ProductType.CNC:
      return kite.PRODUCT_CNC
    return None 
  def convertToBrokerOrderType(self, orderType):
    kite = self.brokerHandle
    if orderType == OrderType.LIMIT:
      return kite.ORDER_TYPE_LIMIT
    elif orderType == OrderType.MARKET:
      return kite.ORDER_TYPE_MARKET
    elif orderType == OrderType.SL_MARKET:
      return kite.ORDER_TYPE_SLM
    elif orderType == OrderType.SL_LIMIT:
      return kite.ORDER_TYPE_SL
```

ZerodhaOrderManager_part2 continues the Zerodha-specific order-management responsibilities by providing the cancel operation, a reconciliation pass that pulls the broker order book and updates internal Order objects, and helper conversions that map framework enums into Zerodha API constants. When an order cancellation is requested the code uses the broker handle obtained from BaseOrderManager (the same brokerHandle used by placeOrder in ZerodhaOrderManager_part1) to invoke the broker cancel flow, logs success or failure, and updates the Order’s lastOrderUpdateTimestamp using Utils.getEpoch so the framework knows when the last broker-side change occurred; failures are logged and re-raised so callers like TradeManager can handle them. The fetchAndUpdateAllOrderDetails routine asks the broker handle for the full orders list, logs the result size, then iterates the returned broker orders and matches them to the in-memory Order instances by comparing the broker order id; for each match it copies quantity, filled and pending quantities, status, price, trigger price and average price into the Order object, applies the business rule that a cancelled order with non-zero fills should be considered completed, increments an update counter and logs a summary — this keeps the TradeManager’s view of entry/SL/target orders in sync with Zerodha so lifecycle actions (for example those orchestrated by TradeManager.trackAndUpdateAllTrades) operate on current state. The convertToBrokerProductType and convertToBrokerOrderType helpers translate the framework ProductType and OrderType enums into the specific constants expected by the Zerodha client; these mappings are used upstream by placeOrder and modifyOrder to ensure the API receives broker-native values. All of these methods rely on the brokerHandle set by BaseOrderManager and follow the same mapping pattern as convertToBrokerDirection implemented in ZerodhaOrderManager_part3 and the placement/modify logic in ZerodhaOrderManager_part1, keeping the Zerodha adapter consistent in how it converts framework intent into broker API calls and how it reconciles broker responses back into framework models.

```python
# file path: src/ordermgmt/ZerodhaOrderManager.py
    return None
  def convertToBrokerDirection(self, direction):
    kite = self.brokerHandle
    if direction == Direction.LONG:
      return kite.TRANSACTION_TYPE_BUY
    elif direction == Direction.SHORT:
      return kite.TRANSACTION_TYPE_SELL
    return None
```

ZerodhaOrderManager_part3 provides the broker-specific translation for trade direction so the rest of the framework can remain broker-agnostic. When a strategy or the TradeManager produces an OrderInputParams with a framework Direction (for example when placeOrder, modifyOrder, modifyOrderToMarket or cancelOrder need to call the broker), convertToBrokerDirection grabs the Zerodha session handle that BaseOrderManager initializes via Controller and maps the framework LONG direction to Zerodha’s buy transaction constant and the framework SHORT direction to Zerodha’s sell transaction constant; if the direction is neither LONG nor SHORT it yields None. This method is one of the conversion helpers alongside convertToBrokerProductType and convertToBrokerOrderType, implementing the adapter pattern so high-level order logic can call a single API and have the correct broker-specific fields filled before interacting with the Zerodha API.

```python
# file path: src/trademgmt/TradeManager.py
        exit = TradeManager.symbolToCMPMap[trade.tradingSymbol]
        TradeManager.setTradeToCompleted(trade, exit, TradeExitReason.TARGET_CANCELLED)
        TradeManager.cancelSLOrder(trade)
  @staticmethod
  def placeSLOrder(trade):
    oip = OrderInputParams(trade.tradingSymbol)
    oip.direction = Direction.SHORT if trade.direction == Direction.LONG else Direction.LONG 
    oip.productType = trade.productType
    oip.orderType = OrderType.SL_MARKET
    oip.triggerPrice = trade.stopLoss
    oip.qty = trade.qty
    if trade.isFutures == True or trade.isOptions == True:
      oip.isFnO = True
    try:
      trade.slOrder = TradeManager.getOrderManager().placeOrder(oip)
    except Exception as e:
      logging.error('TradeManager: Failed to place SL order for tradeID %s: Error => %s', trade.tradeID, str(e))
      return False
    logging.info('TradeManager: Successfully placed SL order %s for tradeID %s', trade.slOrder.orderId, trade.tradeID)
    return True
  @staticmethod
  def placeTargetOrder(trade, isMarketOrder = False):
    oip = OrderInputParams(trade.tradingSymbol)
    oip.direction = Direction.SHORT if trade.direction == Direction.LONG else Direction.LONG
    oip.productType = trade.productType
    oip.orderType = OrderType.MARKET if isMarketOrder == True else OrderType.LIMIT
    oip.price = 0 if isMarketOrder == True else trade.target
    oip.qty = trade.qty
    if trade.isFutures == True or trade.isOptions == True:
      oip.isFnO = True
    try:
      trade.targetOrder = TradeManager.getOrderManager().placeOrder(oip)
    except Exception as e:
      logging.error('TradeManager: Failed to place Target order for tradeID %s: Error => %s', trade.tradeID, str(e))
      return False
    logging.info('TradeManager: Successfully placed Target order %s for tradeID %s', trade.targetOrder.orderId, trade.tradeID)
    return True
  @staticmethod
  def cancelEntryOrder(trade):
    if trade.entryOrder == None:
      return
    if trade.entryOrder.orderStatus == OrderStatus.CANCELLED:
      return
    try:
      TradeManager.getOrderManager().cancelOrder(trade.entryOrder)
    except Exception as e:
      logging.error('TradeManager: Failed to cancel Entry order %s for tradeID %s: Error => %s', trade.entryOrder.orderId, trade.tradeID, str(e))
    logging.info('TradeManager: Successfully cancelled Entry order %s for tradeID %s', trade.entryOrder.orderId, trade.tradeID)
  @staticmethod
  def cancelSLOrder(trade):
    if trade.slOrder == None:
      return
    if trade.slOrder.orderStatus == OrderStatus.CANCELLED:
      return
    try:
      TradeManager.getOrderManager().cancelOrder(trade.slOrder)
    except Exception as e:
      logging.error('TradeManager: Failed to cancel SL order %s for tradeID %s: Error => %s', trade.slOrder.orderId, trade.tradeID, str(e))
    logging.info('TradeManager: Successfully cancelled SL order %s for tradeID %s', trade.slOrder.orderId, trade.tradeID)
  @staticmethod
```

Within the trade lifecycle controller, the methods in TradeManager_part5 are the concrete helpers that translate a Trade object's intended exit behavior into actual broker orders and bookkeeping updates, tying strategy intent to the Zerodha order manager and the Trade object's fields. When a target order is observed to be cancelled externally, the code pulls the latest market price for the trade's symbol from TradeManager.symbolToCMPMap, marks the trade completed by delegating to TradeManager.setTradeToCompleted (defined in TradeManager_part6) with the exit reason set to TARGET_CANCELLED, and then attempts to cancel any standing stop-loss order. The placeSLOrder method builds an OrderInputParams instance for the trade's symbol, flips the direction relative to the trade's entry direction, sets product type and an SL_MARKET order with the trade's stopLoss as the trigger, assigns the trade quantity and marks the order as F&O when appropriate, then hands the parameters to the active order manager via TradeManager.getOrderManager().placeOrder; it records the returned order onto trade.slOrder, logs success, and returns a boolean indicating placement outcome, logging an error on exceptions. The placeTargetOrder method follows the same pattern for the profit target: it prepares OrderInputParams with the opposite direction, sets the product type and either a MARKET or LIMIT order depending on the isMarketOrder flag, assigns price (zero for market) and quantity, marks F&O when needed, places the order through the order manager, stores the resulting order on trade.targetOrder, and returns success or failure while logging outcomes. cancelEntryOrder and cancelSLOrder are defensive cancel helpers that first guard against a missing order or an already cancelled state, then invoke the order manager's cancel operation, catching and logging exceptions and logging successful cancellations; these methods update nothing on the Trade object themselves beyond relying on the order manager to mutate order status fields that the rest of TradeManager's tracking routines (such as trackEntryOrder and trackSLOrder from TradeManager_part4) will observe. Overall, these helpers are the mid-layer that map trade-level exit semantics into OrderInputParams and cancellations, coordinate with ZerodhaOrderManager conversion routines for direction/productType, and update the Trade object's entry/SL/target order references so the lifecycle tracking logic can drive state transitions and P&L calculation.

```python
# file path: src/ordermgmt/OrderInputParams.py
from models.Segment import Segment
from models.ProductType import ProductType
```

OrderInputParams brings in the Segment and ProductType model classes so it can tag, validate and normalize the market segment and product-class of every order it encapsulates; these are the canonical enums the rest of the engine expects when TradeManager builds an order request. Using Segment lets OrderInputParams indicate whether an instrument/order belongs to equity, derivative, or other exchange segments, and using ProductType lets it express the brokerage product category (for example intraday versus delivery) so downstream order managers receive a consistent, typed payload. This mirrors the pattern you’ve already seen elsewhere—ProductType is imported and reused by BaseOrderManager, ZerodhaOrderManager and TradeManager_part6—because the models package centralizes these domain enums for use across connectivity, order-management and lifecycle layers.

```python
# file path: src/ordermgmt/OrderInputParams.py
class OrderInputParams:
  def __init__(self, tradingSymbol):
    self.exchange = "NSE" 
    self.isFnO = False
    self.segment = Segment.EQUITY 
    self.productType = ProductType.MIS 
    self.tradingSymbol = tradingSymbol
    self.direction = ""
    self.orderType = "" 
    self.qty = 0
    self.price = 0
    self.triggerPrice = 0 
  def __str__(self):
    return "symbol=" + str(self.tradingSymbol) + ", exchange=" + self.exchange \
      + ", productType=" + self.productType + ", segment=" + self.segment \
      + ", direction=" + self.direction + ", orderType=" + self.orderType \
      + ", qty=" + str(self.qty) + ", price=" + str(self.price) + ", triggerPrice=" + str(self.triggerPrice) \
      + ", isFnO=" + str(self.isFnO)
```

OrderInputParams is the small, standardized data container TradeManager uses to assemble and hand a fully-specified order to the order-management layer. When a TradeManager flow such as trackAndUpdateAllTrades or the place/cancel flows in TradeManager_part3 and TradeManager_part5 needs to submit or modify an order, it creates an OrderInputParams instance with the instrument identifier and then fills in the actionable fields (direction, orderType, qty, price, triggerPrice) before passing it to the concrete OrderManager (for example ZerodhaOrderManager). The constructor establishes framework-wide defaults so callers only need to supply the values that vary per order: the exchange defaults to the engine’s primary market, the instrument segment defaults to the equity segment, the product defaults to intraday MIS, and the instrument-level flags mark it as not F&O by default; numeric fields start at zero and text fields start empty. Those defaults reflect the trading platform’s typical use-case and keep TradeManager code concise and consistent. The string representation method produces a compact, human-readable summary of all of those fields so the rest of the system can log or display a single-line snapshot of the intended order before or after submission. OrderInputParams therefore plays the leaf utility role of normalizing and carrying order intent into the order manager and works alongside the Order, OrderModifyParams, and Trade classes: Order consumes an OrderInputParams instance to populate persistent order state, OrderModifyParams is used for later edits, and Trade holds the broader lifecycle and strategy context around these parameter objects.

```python
# file path: src/strategies/BNFORB30Min.py
import logging
from datetime import datetime
from instruments.Instruments import Instruments
from models.Direction import Direction
from models.ProductType import ProductType
from strategies.BaseStrategy import BaseStrategy
from utils.Utils import Utils
from trademgmt.Trade import Trade
from trademgmt.TradeManager import TradeManager
```

These imports wire the BNFORB30Min strategy into the surrounding framework by bringing in the runtime tools, domain models, base strategy contract, utility helpers, and the trade lifecycle primitives the strategy needs to generate signals and execute them. Logging provides the same consistent runtime diagnostics other core pieces produce so the strategy can emit lifecycle and debug messages. datetime supplies the time primitives used to build the 30‑minute opening range boundaries and timestamp trades and decisions. Instruments gives access to instrument metadata and selection helpers so the strategy can resolve the Bank Nifty underlying, strikes and expiry that the ORB logic targets. Direction and ProductType supply the domain enums the strategy uses when deciding buy versus sell and which product type to request from the broker. BaseStrategy is the framework contract this class implements/extends so the strategy can plug into the engine’s lifecycle, receive normalized market data and expose its signals. Utils provides small helpers the strategy uses for common operations (time rounding, price math, etc.). Trade is the local model used to represent a candidate or live position with its metadata, and TradeManager is the orchestration layer the strategy calls to submit, modify or exit orders and to query lifecycle state. This set follows the same pattern seen in other strategy files—most import logging, BaseStrategy, Utils, Trade and TradeManager—while including Instruments and Direction here because BNFORB30Min needs explicit instrument selection and directional enums rather than a quotes adapter.

```python
# file path: src/trademgmt/TradeManager.py
        continue
      if tr.strategy != strategy:
        continue
      if tr.direction != direction:
        continue
      trade = tr
      break
    return trade
  @staticmethod
  def executeTrade(trade):
    logging.info('TradeManager: Execute trade called for %s', trade)
    trade.initialStopLoss = trade.stopLoss
    oip = OrderInputParams(trade.tradingSymbol)
    oip.direction = trade.direction
    oip.productType = trade.productType
    oip.orderType = OrderType.MARKET if trade.placeMarketOrder == True else OrderType.LIMIT
    oip.price = trade.requestedEntry
    oip.qty = trade.qty
    if trade.isFutures == True or trade.isOptions == True:
      oip.isFnO = True
    try:
      trade.entryOrder = TradeManager.getOrderManager().placeOrder(oip)
    except Exception as e:
      logging.error('TradeManager: Execute trade failed for tradeID %s: Error => %s', trade.tradeID, str(e))
      return False
    logging.info('TradeManager: Execute trade successful for %s and entryOrder %s', trade, trade.entryOrder)
    return True
  @staticmethod
  def fetchAndUpdateAllTradeOrders():
    allOrders = []
    for trade in TradeManager.trades:
      if trade.entryOrder != None:
        allOrders.append(trade.entryOrder)
      if trade.slOrder != None:
        allOrders.append(trade.slOrder)
      if trade.targetOrder != None:
        allOrders.append(trade.targetOrder)
    TradeManager.getOrderManager().fetchAndUpdateAllOrderDetails(allOrders)
  @staticmethod
  def trackAndUpdateAllTrades():
    for trade in TradeManager.trades:
      if trade.tradeState == TradeState.ACTIVE:
        TradeManager.trackEntryOrder(trade)
        TradeManager.trackSLOrder(trade)
        TradeManager.trackTargetOrder(trade)
        if trade.intradaySquareOffTimestamp != None:
          nowEpoch = Utils.getEpoch()
          if nowEpoch >= trade.intradaySquareOffTimestamp:
            TradeManager.squareOffTrade(trade, TradeExitReason.SQUARE_OFF)
  @staticmethod
  def trackEntryOrder(trade):
    if trade.tradeState != TradeState.ACTIVE:
      return
    if trade.entryOrder == None:
      return
    if trade.entryOrder.orderStatus == OrderStatus.CANCELLED or trade.entryOrder.orderStatus == OrderStatus.REJECTED:
      trade.tradeState = TradeState.CANCELLED
    trade.filledQty = trade.entryOrder.filledQty
    if trade.filledQty > 0:
      trade.entry = trade.entryOrder.averagePrice
```

TradeManager_part3 sits in the execution and lifecycle glue layer of sdoosa-algo-trade-python-master_cleaned and implements the runtime actions that turn a strategy-generated Trade into a broker order and then keep that Trade updated as broker state changes. When executeTrade is invoked for a Trade, it first snapshots the current stop loss into initialStopLoss (used later by the trailing SL logic found in TradeManager_part4), then constructs an OrderInputParams for the trade's instrument, fills canonical fields such as direction, productType, quantity and price, chooses a MARKET or LIMIT order type based on the trade's placeMarketOrder flag, and marks the request as FnO when the Trade represents futures or options. That OrderInputParams is handed to the order manager obtained via TradeManager.getOrderManager(), which delegates to the broker-specific placeOrder implementation (ZerodhaOrderManager); executeTrade captures the returned Order object into trade.entryOrder, logs success, and returns a boolean outcome, logging and returning failure if the placeOrder call throws. fetchAndUpdateAllTradeOrders is a polling/refresh helper that walks the in-memory TradeManager.trades list, collects any entry, SL and target Order objects present on each Trade, and asks the order manager to fetch and update details for that aggregated list so Order objects kept on Trades reflect the broker's latest state. trackAndUpdateAllTrades is the periodic tracker that iterates active trades and drives three sub-flows: it calls trackEntryOrder to reconcile entry order fills and statuses, calls trackSLOrder (the SL placement and completion logic implemented in TradeManager_part4) and trackTargetOrder (the exit/target logic implemented

```python
# file path: src/trademgmt/TradeManager.py
    trade.cmp = TradeManager.symbolToCMPMap[trade.tradingSymbol]
    Utils.calculateTradePnl(trade)
  @staticmethod
  def trackSLOrder(trade):
    if trade.tradeState != TradeState.ACTIVE:
      return
    if trade.stopLoss == 0: 
      return
    if trade.slOrder == None:
      TradeManager.placeSLOrder(trade)
    else:
      if trade.slOrder.orderStatus == OrderStatus.COMPLETE:
        exit = trade.slOrder.averagePrice
        exitReason = TradeExitReason.SL_HIT if trade.initialStopLoss == trade.stopLoss else TradeExitReason.TRAIL_SL_HIT
        TradeManager.setTradeToCompleted(trade, exit, exitReason)
        TradeManager.cancelTargetOrder(trade)
      elif trade.slOrder.orderStatus == OrderStatus.CANCELLED:
        logging.error('SL order %s for tradeID %s cancelled outside of Algo. Setting the trade as completed with exit price as current market price.', trade.slOrder.orderId, trade.tradeID)
        exit = TradeManager.symbolToCMPMap[trade.tradingSymbol]
        TradeManager.setTradeToCompleted(trade, exit, TradeExitReason.SL_CANCELLED)
        TradeManager.cancelTargetOrder(trade)
      else:
        TradeManager.checkAndUpdateTrailSL(trade)
  @staticmethod
  def checkAndUpdateTrailSL(trade):
    strategyInstance = TradeManager.strategyToInstanceMap[trade.strategy]
    if strategyInstance == None:
      return
    newTrailSL = strategyInstance.getTrailingSL(trade)
    updateSL = False
    if newTrailSL > 0:
      if trade.direction == Direction.LONG and newTrailSL > trade.stopLoss:
        updateSL = True
      elif trade.direction == Direction.SHORT and newTrailSL < trade.stopLoss:
        updateSL = True
    if updateSL == True:
      omp = OrderModifyParams()
      omp.newTriggerPrice = newTrailSL
      try:
        oldSL = trade.stopLoss
        TradeManager.getOrderManager().modifyOrder(trade.slOrder, omp)
        logging.info('TradeManager: Trail SL: Successfully modified stopLoss from %f to %f for tradeID %s', oldSL, newTrailSL, trade.tradeID)
        trade.stopLoss = newTrailSL 
      except Exception as e:
        logging.error('TradeManager: Failed to modify SL order for tradeID %s orderId %s: Error => %s', trade.tradeID, trade.slOrder.orderId, str(e))
  @staticmethod
  def trackTargetOrder(trade):
    if trade.tradeState != TradeState.ACTIVE:
      return
    if trade.target == 0: 
      return
    if trade.targetOrder == None:
      TradeManager.placeTargetOrder(trade)
    else:
      if trade.targetOrder.orderStatus == OrderStatus.COMPLETE:
        exit = trade.targetOrder.averagePrice
        TradeManager.setTradeToCompleted(trade, exit, TradeExitReason.TARGET_HIT)
        TradeManager.cancelSLOrder(trade)
      elif trade.targetOrder.orderStatus == OrderStatus.CANCELLED:
        logging.error('Target order %s for tradeID %s cancelled outside of Algo. Setting the trade as completed with exit price as current market price.', trade.targetOrder.orderId, trade.tradeID)
```

TradeManager_part4 contains the runtime monitoring and lifecycle logic that keeps active trades synchronized with broker orders and applies basic exit/risk rules. trackSLOrder first guards that the Trade is ACTIVE and has a nonzero stopLoss; if there is no slOrder yet it delegates to placeSLOrder (covered in TradeManager_part5) to create the broker SL order. If an slOrder exists it examines the orderStatus: when the order is COMPLETE it treats the trade as exited using the slOrder averagePrice and sets the exitReason to SL_HIT or TRAIL_SL_HIT depending on whether the stopLoss has been trailed since entry, then marks the Trade completed and cancels the target via cancelTargetOrder (from TradeManager_part6). If the slOrder was CANCELLED externally it logs an error, uses the current market price from symbolToCMPMap as the exit, marks the Trade completed with SL_CANCELLED, and cancels the target. For any other SL order state it invokes checkAndUpdateTrailSL to potentially move the stop loss. checkAndUpdateTrailSL looks up the strategy instance from strategyToInstanceMap and, if present, asks the strategy for a new trailing stop via getTrailingSL (BaseStrategy.getTrailingSL). It only proceeds when the returned newTrailSL is a positive, “better” SL (higher for a LONG, lower for a SHORT). In that case it constructs an OrderModifyParams with the new trigger price and calls getOrderManager().modifyOrder to update the broker SL; on success it updates trade.stopLoss and logs the change, and on failure it logs an error. trackTargetOrder mirrors the

```python
# file path: src/ordermgmt/OrderModifyParams.py
class OrderModifyParams:
  def __init__(self):
    self.newPrice = 0
    self.newTriggerPrice = 0 
    self.newQty = 0
    self.newOrderType = None 
  def __str__(self):
    return "newPrice=" + str(self.newPrice) + ", newTriggerPrice=" + str(self.newTriggerPrice) \
      + ", newQty=" + str(self.newQty) + ", newOrderType=" + str(self.newOrderType)
```

OrderModifyParams is the tiny, purpose-built data container TradeManager_part4 uses whenever the engine needs to change an existing order on the wire. In the order lifecycle story, a TradeManager routine such as checkAndUpdateTrailSL computes new trade parameters (often using a strategy-provided value from BaseStrategy.getTrailingSL) and then creates an OrderModifyParams instance to package those new values — newPrice, newTriggerPrice, newQty and newOrderType — with sensible defaults (zeros or None) so the rest of the system has a predictable object to work with. Once populated, TradeManager hands the OrderModifyParams to the order-management layer, which in the Zerodha adapter is consumed by ZerodhaOrderManager.modifyOrder or modifyOrderToMarket; those adapters inspect the fields and translate only the non-default values down to the broker API. OrderModifyParams follows the same small DTO pattern you’ve seen in OrderInputParams and Order: it exists to normalize and label a specific set of broker-facing parameters and to provide a readable string form for logging and debugging, enabling clear, consistent handoff from trade lifecycle logic to the execution layer.

```python
# file path: src/trademgmt/TradeManager.py
  def convertJSONToTrade(jsonData):
    trade = Trade(jsonData['tradingSymbol'])
    trade.tradeID = jsonData['tradeID']
    trade.strategy = jsonData['strategy']
    trade.direction = jsonData['direction']
    trade.productType = jsonData['productType']
    trade.isFutures = jsonData['isFutures']
    trade.isOptions = jsonData['isOptions']
    trade.optionType = jsonData['optionType']
    trade.placeMarketOrder = jsonData['placeMarketOrder']
    trade.intradaySquareOffTimestamp = jsonData['intradaySquareOffTimestamp']
    trade.requestedEntry = jsonData['requestedEntry']
    trade.entry = jsonData['entry']
    trade.qty = jsonData['qty']
    trade.filledQty = jsonData['filledQty']
    trade.initialStopLoss = jsonData['initialStopLoss']
    trade.stopLoss = jsonData['stopLoss']
    trade.target = jsonData['target']
    trade.cmp = jsonData['cmp']
    trade.tradeState = jsonData['tradeState']
    trade.timestamp = jsonData['timestamp']
    trade.createTimestamp = jsonData['createTimestamp']
    trade.startTimestamp = jsonData['startTimestamp']
    trade.endTimestamp = jsonData['endTimestamp']
    trade.pnl = jsonData['pnl']
    trade.pnlPercentage = jsonData['pnlPercentage']
    trade.exit = jsonData['exit']
    trade.exitReason = jsonData['exitReason']
    trade.exchange = jsonData['exchange']
    trade.entryOrder = TradeManager.convertJSONToOrder(jsonData['entryOrder'])
    trade.slOrder = TradeManager.convertJSONToOrder(jsonData['slOrder'])
    trade.targetOrder = TradeManager.convertJSONToOrder(jsonData['targetOrder'])
    return trade
  @staticmethod
  def convertJSONToOrder(jsonData):
    if jsonData == None:
      return None
    order = Order()
    order.tradingSymbol = jsonData['tradingSymbol']
    order.exchange = jsonData['exchange']
    order.productType = jsonData['productType']
    order.orderType = jsonData['orderType']
    order.price = jsonData['price']
    order.triggerPrice = jsonData['triggerPrice']
    order.qty = jsonData['qty']
    order.orderId = jsonData['orderId']
    order.orderStatus = jsonData['orderStatus']
    order.averagePrice = jsonData['averagePrice']
    order.filledQty = jsonData['filledQty']
    order.pendingQty = jsonData['pendingQty']
    order.orderPlaceTimestamp = jsonData['orderPlaceTimestamp']
    order.lastOrderUpdateTimestamp = jsonData['lastOrderUpdateTimestamp']
    order.message = jsonData['message']
    return order
  @staticmethod
  def getLastTradedPrice(tradingSymbol):
    return TradeManager.symbolToCMPMap[tradingSymbol]
```

Within the trade lifecycle controller, convertJSONToTrade is the deserialization path TradeManager uses to rebuild a Trade instance from persisted JSON so the engine can resume tracking and managing that trade after a restart or when reloading state. It begins by creating a Trade with the trading symbol so the instance gets the defaults defined by Trade.__init__, then overwrites the instance’s identifying, configuration and runtime fields with values from the JSON — things like tradeID, strategy, direction, product and instrument flags, requestedEntry/entry, quantities and filled quantities, stopLoss/initialStopLoss/target, timestamps, P&L fields and exit metadata — effectively reconstructing the exact runtime snapshot the lifecycle controller needs to continue processing. For the nested order objects the deserializer delegates to convertJSONToOrder, which returns None for absent orders or instantiates an Order and populates its exchange, productType, orderType, price/trigger, quantities, ids, status, average/fill info and timestamps so the order-management layer can resume reconciliation; convertJSONToOrder therefore mirrors how Order.__init__ normally seeds an order when created from OrderInputParams but populates every runtime field from the stored JSON. The getLastTradedPrice helper is a thin accessor that returns the current CMP from TradeManager.symbolToCMPMap for a given trading symbol, which strategies and tracking routines use when recalculating trailing stops or P&L after state load. Together these methods let TradeManager load persisted trades, restore their linked Order state so ZerodhaOrderManager and the rest of the lifecycle logic (as implemented in the other TradeManager parts) can continue to place/modify/cancel or track fills, and provide quick access to the last traded price for downstream decision points.

```python
# file path: src/utils/Utils.py
  def isTodayOneDayBeforeWeeklyExpiryDay():
    expiryDate = Utils.getWeeklyExpiryDayDate()
    todayDate = Utils.getTimeOfToDay(0, 0, 0)
    if expiryDate - timedelta(days=1) == todayDate:
      return True  
    return False
  @staticmethod
  def getNearestStrikePrice(price, nearestMultiple = 50):
    inputPrice = int(price)
    remainder = int(inputPrice % nearestMultiple)
    if remainder < int(nearestMultiple / 2):
      return inputPrice - remainder
    else:
      return inputPrice + (nearestMultiple - remainder)
```

Utils_part4 contains two small, strategy-facing helpers that sit in the shared utilities layer and are used by the options strategies to make expiry and strike decisions. isTodayOneDayBeforeWeeklyExpiryDay asks Utils for the canonical weekly expiry date (via getWeeklyExpiryDayDate, which we covered earlier) and compares that expiry to the current calendar day at midnight (via getTimeOfToDay); it returns a boolean indicating whether today is exactly one calendar day before the weekly expiry. That boolean is used by option-selling and straddle strategies to alter behavior around expiry (for example, to avoid initiating fresh FnO positions the day before expiry). getNearestStrikePrice converts a raw numeric price into the nearest valid option strike given a configurable strike interval (defaulting to fifty). It does this by coercing the input to an integer, computing the remainder against the strike interval, and choosing the lower multiple when the remainder is less than half the interval or the higher multiple otherwise, producing a deterministic nearest-strike value that downstream code can pass to symbol-preparation helpers (such as prepareWeeklyOptionsSymbol) when generating option trading symbols. Together these helpers keep expiry-awareness and strike-normalization consistent across OptionSelling, ShortStraddleBNF and other strategy code that needs to select tradable option strikes or gate trades near expiry.

```python
# file path: src/restapis/StartAlgoAPI.py
from flask.views import MethodView
import json
import logging
import threading
from config.Config import getSystemConfig
from core.Algo import Algo
```

The imports set up StartAlgoAPI as a web-facing control that can load runtime configuration and kick off the algorithmic runtime in the background. MethodView supplies the Flask class-based view foundation so StartAlgoAPI can be mounted as a REST endpoint; json is used to decode incoming payloads and produce JSON responses; logging provides runtime diagnostics and event messages from the endpoint; threading allows the endpoint to spawn the algorithm run off the request thread so the HTTP call can return immediately while the trading runtime continues asynchronously. getSystemConfig from config.Config is the runtime configuration loader and validator that prepares broker credentials, strategy parameters, risk limits and environment-specific settings before any components are instantiated. Algo from core.Algo is the orchestration object that composes the strategy, market data, order manager and trade lifecycle pieces and actually initializes and starts the trading run — this is the piece StartAlgoAPI hands control to after configuration is validated. Compared to other controllers in the codebase that import Controller to handle request-level coordination, StartAlgoAPI differs because it needs direct access to configuration and the Algo orchestration layer and uses threading to launch a long-lived runtime rather than simply delegating to a Controller for synchronous request handling.

```python
# file path: src/restapis/StartAlgoAPI.py
class StartAlgoAPI(MethodView):
  def post(self):
    x = threading.Thread(target=Algo.startAlgo)
    x.start()
    systemConfig = getSystemConfig()
    homeUrl = systemConfig['homeUrl'] + '?algoStarted=true'
    logging.info('Sending redirect url %s in response', homeUrl)
    respData = { 'redirect': homeUrl }
    return json.dumps(respData)
```

StartAlgoAPI is the HTTP entry that triggers a live algo run: when its post handler is invoked it launches the framework’s startup sequence in the background and immediately returns a small JSON response that instructs the UI to redirect. Concretely, the handler spawns a new thread that invokes Algo.startAlgo so the request thread does not block while the engine initializes; that call to Algo.startAlgo will perform the actual initialization steps you’ve seen elsewhere (checking the isAlgoRunning guard, fetching instruments, spinning up the TradeManager thread and the strategy thread, and setting the running flag). After kicking off the background start, the handler loads runtime configuration by calling getSystemConfig (which reads and parses the system JSON file), builds a redirect URL by appending the algoStarted flag to the configured homeUrl, logs that redirect, and returns a JSON payload containing that URL. The returned redirect ties into HomeAPI’s logic so the web UI will render the “algo started” view when the browser follows the redirect. In short, StartAlgoAPI coordinates the non-blocking launch of the trading engine and communicates that state change back to the frontend via a config-derived redirect.

```python
# file path: src/strategies/BNFORB30Min.py
class BNFORB30Min(BaseStrategy):
  __instance = None
  @staticmethod
  def getInstance(): 
    if BNFORB30Min.__instance == None:
      BNFORB30Min()
    return BNFORB30Min.__instance
  def __init__(self):
    if BNFORB30Min.__instance != None:
      raise Exception("This class is a singleton!")
    else:
      BNFORB30Min.__instance = self
    super().__init__("BNFORB30Min")
    self.productType = ProductType.MIS
    self.symbols = []
    self.slPercentage = 0
    self.targetPercentage = 0
    self.startTimestamp = Utils.getTimeOfToDay(9, 45, 0) 
    self.stopTimestamp = Utils.getTimeOfToDay(14, 30, 0) 
    self.squareOffTimestamp = Utils.getTimeOfToDay(15, 0, 0) 
    self.capital = 100000 
    self.leverage = 0
    self.maxTradesPerDay = 1 
    self.isFnO = True 
    self.capitalPerSet = 100000 
  def process(self):
    now = datetime.now()
    processEndTime = Utils.getTimeOfToDay(9, 50, 0)
    if now < self.startTimestamp:
      return
    if now > processEndTime:
      return
    if len(self.trades) >= 2:
      return
    symbol = Utils.prepareMonthlyExpiryFuturesSymbol('BANKNIFTY')
    quote = self.getQuote(symbol)
    if quote == None:
        logging.error('%s: Could not get quote for %s', self.getName(), symbol)
        return
    logging.info('%s: %s => lastTradedPrice = %f', self.getName(), symbol, quote.lastTradedPrice)
    self.generateTrade(symbol, Direction.LONG, quote.high, quote.low)
    self.generateTrade(symbol, Direction.SHORT, quote.high, quote.low)
  def generateTrade(self, tradingSymbol, direction, high, low):
    trade = Trade(tradingSymbol)
    trade.strategy = self.getName()
    trade.isFutures = True
    trade.direction = direction
    trade.productType = self.productType
    trade.placeMarketOrder = True
    trade.requestedEntry = high if direction == Direction.LONG else low
    trade.timestamp = Utils.getEpoch(self.startTimestamp) 
    numLots = self.calculateLotsPerTrade()
    isd = Instruments.getInstrumentDataBySymbol(tradingSymbol) 
    trade.qty = isd['lot_size'] * numLots
    trade.stopLoss = low if direction == Direction.LONG else high
    slDiff = high - low
    if direction == 'LONG':
      trade.target = Utils.roundToNSEPrice(trade.requestedEntry + 1.5 * slDiff)
    else:
      trade.target = Utils.roundToNSEPrice(trade.requestedEntry - 1.5 * slDiff)
```

BNFORB30Min_part1 implements the signal-generation half of the Bank Nifty Opening Range Breakout strategy as a singleton subclass of BaseStrategy, so only one instance runs and it inherits the framework defaults for strategy wiring. Its constructor configures the strategy-specific parameters: it sets the productType to intraday MIS, an empty symbol list (it resolves BANKNIFTY dynamically), stop/start/square-off times using Utils.getTimeOfToDay, capital and sizing fields, maxTradesPerDay and other flags that tell the engine this is an FnO futures strategy. The process method is the runtime entry: it restrict

```python
# file path: src/strategies/OptionSelling.py
import logging
from datetime import datetime
from instruments.Instruments import Instruments
from models.Direction import Direction
from models.ProductType import ProductType
from strategies.BaseStrategy import BaseStrategy
from utils.Utils import Utils
from trademgmt.Trade import Trade
from trademgmt.TradeManager import TradeManager
```

The imports wire the OptionSelling strategy into the rest of the engine by bringing in the runtime, domain and lifecycle pieces it needs to generate option-selling signals, build Trade objects and hand them off for execution. Logging provides strategy-level observability so runtime decisions and errors are recorded. Datetime supplies timestamping and time-based decision points used by expiry and intraday rules. Instruments gives access to instrument metadata and helpers for selecting option strikes and expiries, which is central to any options-selling logic. Direction and ProductType are the canonical enums the framework uses to tag trade intent and order-class so trades created by OptionSelling conform to the same segment/product conventions used throughout the engine (recall OrderInputParams relies on these canonical enums). BaseStrategy is the pluggable strategy contract OptionSelling implements so it can be wired into the strategy runner. Utils supplies shared helper functions (rounding, price checks, small calculations and formatting) that keep strategy code concise. Trade is the domain model the strategy instantiates to represent an open position and its lifecycle metadata. TradeManager is the lifecycle controller the strategy delegates to when it wants to place, modify or exit trades (this aligns with the TradeManager_part5 helpers you reviewed earlier). Compared to the other similar import blocks, this set follows the common pattern of including logging, datetime, BaseStrategy, Utils and the domain enums, but differs by explicitly pulling in Instruments and the Trade class here because OptionSelling actively selects option instruments and constructs Trade objects itself, whereas some other strategies only referenced Quotes or omitted direct Trade instantiation.

```python
# file path: src/strategies/OptionSelling.py
class OptionSelling(BaseStrategy):
  __instance = None
  @staticmethod
  def getInstance(): 
    if OptionSelling.__instance == None:
      OptionSelling()
    return OptionSelling.__instance
  def __init__(self):
    if OptionSelling.__instance != None:
      raise Exception("This class is a singleton!")
    else:
      OptionSelling.__instance = self
    super().__init__("OptionSelling")
    self.productType = ProductType.MIS
    self.symbols = []
    self.slPercentage = 50
    self.targetPercentage = 0
    self.startTimestamp = Utils.getTimeOfToDay(9, 30, 0) 
    self.stopTimestamp = Utils.getTimeOfToDay(14, 30, 0) 
    self.squareOffTimestamp = Utils.getTimeOfToDay(15, 15, 0) 
    self.capital = 100000 
    self.leverage = 0
    self.maxTradesPerDay = 2 
    self.isFnO = True 
    self.capitalPerSet = 100000 
  def canTradeToday(self):
    if Utils.isTodayOneDayBeforeWeeklyExpiryDay() == True:
      logging.info('%s: Today is one day before weekly expiry date hence going to trade this strategy', self.getName())
      return True
    if Utils.isTodayWeeklyExpiryDay() == True:
      logging.info('%s: Today is weekly expiry day hence going to trade this strategy today', self.getName())
      return True
    logging.info('%s: Today is neither day before expiry nor expiry day. Hence NOT going to trade this strategy today', self.getName())
    return False
  def process(self):
    now = datetime.now()
    if now < self.startTimestamp:
      return
    if len(self.trades) >= self.maxTradesPerDay:
      return
    futureSymbol = Utils.prepareMonthlyExpiryFuturesSymbol('NIFTY')
    quote = self.getQuote(futureSymbol)
    if quote == None:
      logging.error('%s: Could not get quote for %s', self.getName(), futureSymbol)
      return
    ATMStrike = Utils.getNearestStrikePrice(quote.lastTradedPrice, 50)
    logging.info('%s: Nifty CMP = %f, ATMStrike = %d', self.getName(), quote.lastTradedPrice, ATMStrike)
    ATMPlus50CESymbol = Utils.prepareWeeklyOptionsSymbol("NIFTY", ATMStrike + 50, 'CE')
    ATMMinus50PESymbol = Utils.prepareWeeklyOptionsSymbol("NIFTY", ATMStrike - 50, 'PE')
    logging.info('%s: ATMPlus50CE = %s, ATMMinus50PE = %s', self.getName(), ATMPlus50CESymbol, ATMMinus50PESymbol)
    self.generateTrades(ATMPlus50CESymbol, ATMMinus50PESymbol)
  def generateTrades(self, ATMPlus50CESymbol, ATMMinus50PESymbol):
    numLots = self.calculateLotsPerTrade()
    quoteATMPlus50CESymbol = self.getQuote(ATMPlus50CESymbol)
    quoteATMMinus50PESymbol = self.getQuote(ATMMinus50PESymbol)
    if quoteATMPlus50CESymbol == None or quoteATMMinus50PESymbol == None:
      logging.error('%s: Could not get quotes for option symbols', self.getName())
      return
    self.generateTrade(ATMPlus50CESymbol, numLots, quoteATMPlus50CESymbol.lastTradedPrice)
    self.generateTrade(ATMMinus50PESymbol, numLots, quoteATMMinus50PESymbol.lastTradedPrice)
```

OptionSelling is a singleton subclass of BaseStrategy that wires the option-selling logic into the framework’s strategy layer and configures the strategy-level parameters that drive trade generation and lifecycle behavior. Its getInstance and __init__ enforce the singleton pattern and call BaseStrategy.__init__ with the strategy name so it inherits the framework defaults, then overrides several fields to tailor this strategy: it sets productType to MIS, enables F&O behavior via isFnO, raises slPercentage to 50, sets capital and capitalPerSet, restricts maxTradesPerDay to 2, and places its active window using Utils.getTimeOfToDay to define startTimestamp, stopTimestamp and squareOffTimestamp. canTradeToday decides whether the strategy should run by consulting the expiry helpers in Utils: it will return true only if today is either the weekly expiry day or the day before weekly expiry (using Utils.isTodayWeeklyExpiryDay and Utils.isTodayOneDayBeforeWeeklyExpiryDay) and logs the decision. process is the runtime entry called periodically by the engine: it first gates execution by time (no action before startTimestamp) and by maxTradesPerDay, then derives the underlying instrument to use for strike calculations by asking Utils.prepareMonthlyExpiryFuturesSymbol for the NIFTY monthly future and fetching its live quote via BaseStrategy.getQuote; if the quote is unavailable it logs an error and stops. With a valid futures quote it computes an ATM strike using Utils.getNearestStrikePrice with a 50-point multiple, logs the computed ATM, builds two weekly option symbols — one 50 points above as a call (CE) and one 50 points below as a put (PE) using Utils.prepareWeeklyOptionsSymbol — and passes those symbols to generateTrades. generateTrades calculates the number of lots via calculateLotsPerTrade, fetches quotes for both option symbols through getQuote, logs and returns if either quote is missing, and then invokes generateTrade for each option with the computed lots and each option’s lastTradedPrice; generateTrade and the strategy’s shouldPlaceTrade logic live in the companion OptionSelling implementation and will handle constructing Trade objects and submitting them to the TradeManager. The overall pattern and lifecycle wiring mirror other singleton strategies such as ShortStraddleBNF, BNFORB30Min and SampleStrategy, but OptionSelling differs in its expiry-aware canTradeToday gating, its use of the monthly futures price on NIFTY to derive ATM strikes with a 50-point offset, and its specific two-leg option-selling generation.

```python
# file path: src/strategies/OptionSelling.py
    logging.info('%s: Trades generated.', self.getName())
  def generateTrade(self, optionSymbol, numLots, lastTradedPrice):
    trade = Trade(optionSymbol)
    trade.strategy = self.getName()
    trade.isOptions = True
    trade.direction = Direction.SHORT 
    trade.productType = self.productType
    trade.placeMarketOrder = True
    trade.requestedEntry = lastTradedPrice
    trade.timestamp = Utils.getEpoch(self.startTimestamp) 
    isd = Instruments.getInstrumentDataBySymbol(optionSymbol) 
    trade.qty = isd['lot_size'] * numLots
    trade.stopLoss = Utils.roundToNSEPrice(trade.requestedEntry + trade.requestedEntry * self.slPercentage / 100)
    trade.target = 0 
    trade.intradaySquareOffTimestamp = Utils.getEpoch(self.squareOffTimestamp)
    TradeManager.addNewTrade(trade)
  def shouldPlaceTrade(self, trade, tick):
    if super().shouldPlaceTrade(trade, tick) == False:
      return False
    return True
```

Within the pluggable strategy layer, OptionSelling_part2 is responsible for turning an option-selling signal into a concrete Trade object and deciding whether that Trade should be sent to the order pipeline. When OptionSelling_part1 determines a candidate option and calls generateTrade, generateTrade constructs a Trade tied to the supplied optionSymbol: it tags the trade with the strategy name, marks it as an options short position, and sets the product type from the strategy instance. The method marks the trade to be placed as a market order and records the requested entry price and an entry timestamp derived from the strategy’s startTimestamp (using Utils.getEpoch). Position sizing is computed by looking up instrument metadata via Instruments.getInstrumentDataBySymbol to obtain the lot size and multiplying by the supplied numLots to set trade.qty. Risk parameters are derived from the strategy configuration: stopLoss is calculated as the requestedEntry plus the configured slPercentage of that entry, rounded to the exchange price grid with Utils.roundToNSEPrice; target is left unset (zero) because this strategy uses no fixed profit target; and an intraday square-off epoch is set from the strategy’s squareOffTimestamp. Once populated, the Trade is handed to TradeManager.addNewTrade so the engine can persist it and start lifecycle handling (order placement, tracking and exits). The shouldPlaceTrade override simply defers to BaseStrategy.shouldPlaceTrade and returns True only if the base class allows placement, adding no additional gating logic; this mirrors the placement behavior used by ShortStraddleBNF where the base checks govern whether a queued trade is eligible to be sent to the broker. Overall, OptionSelling_part2 plays the builder-and-gatekeeper role in the end-to-end flow: it synthesizes trade details from strategy parameters, market prices and instrument metadata, and then registers the trade with the TradeManager for execution and tracking.

```python
# file path: src/strategies/SampleStrategy.py
import logging
from models.Direction import Direction
from models.ProductType import ProductType
from strategies.BaseStrategy import BaseStrategy
from utils.Utils import Utils
from trademgmt.Trade import Trade
from trademgmt.TradeManager import TradeManager
```

The imports set up the minimal scaffolding SampleStrategy needs to generate signals and hand them off into the framework’s execution pipeline. logging provides runtime diagnostics so the strategy can emit info, warnings and errors while running. Direction and ProductType bring in the domain enums that the strategy uses to describe trade side and the product segment when it builds signals and Trade objects. BaseStrategy supplies the common strategy lifecycle and wiring that SampleStrategy extends for signal logic, and Utils gives access to the small, strategy-facing helper routines (the same helpers we looked at in Utils_part4) used for things like expiry and strike decisions. Trade and TradeManager are the trade-domain pieces used by the execution half: Trade is the in-memory trade representation the strategy constructs, and TradeManager is the lifecycle/execution coordinator that the strategy will hand those Trade objects to for order placement, tracking and state transitions. Compared with the similar import lists you’ve seen elsewhere, this set is intentionally focused on strategy and trade lifecycle concerns; other modules add datetime, Quotes or Instruments when they need direct market feed or metadata access, whereas SampleStrategy limits itself to the core models, base class and lifecycle utilities needed to demonstrate the signal → Trade → TradeManager flow within the sdoosa-algo-trade-python-master_cleaned framework.

```python
# file path: src/strategies/SampleStrategy.py
class SampleStrategy(BaseStrategy):
  __instance = None
  @staticmethod
  def getInstance(): 
    if SampleStrategy.__instance == None:
      SampleStrategy()
    return SampleStrategy.__instance
  def __init__(self):
    if SampleStrategy.__instance != None:
      raise Exception("This class is a singleton!")
    else:
      SampleStrategy.__instance = self
    super().__init__("SAMPLE")
    self.productType = ProductType.MIS
    self.symbols = ["SBIN", "INFY", "TATASTEEL", "RELIANCE", "HDFCBANK", "CIPLA"]
    self.slPercentage = 1.1
    self.targetPercentage = 2.2
    self.startTimestamp = Utils.getTimeOfToDay(9, 30, 0) 
    self.stopTimestamp = Utils.getTimeOfToDay(14, 30, 0) 
    self.squareOffTimestamp = Utils.getTimeOfToDay(15, 0, 0) 
    self.capital = 3000 
    self.leverage = 2 
    self.maxTradesPerDay = 3 
    self.isFnO = False 
    self.capitalPerSet = 0 
  def process(self):
    if len(self.trades) >= self.maxTradesPerDay:
      return
    for symbol in self.symbols:
      quote = self.getQuote(symbol)
      if quote == None:
        logging.error('%s: Could not get quote for %s', self.getName(), symbol)
        continue
      longBreakoutPrice = Utils.roundToNSEPrice(quote.close + quote.close * 0.5 / 100)
      shortBreakoutPrice = Utils.roundToNSEPrice(quote.close - quote.close * 0.5 / 100)
      cmp = quote.lastTradedPrice
      logging.info('%s: %s => long = %f, short = %f, CMP = %f', self.getName(), symbol, longBreakoutPrice, shortBreakoutPrice, cmp)
      direction = None
      breakoutPrice = 0
      if cmp > longBreakoutPrice:
        direction = 'LONG'
        breakoutPrice = longBreakoutPrice
      elif cmp < shortBreakoutPrice:
        direction = 'SHORT'
        breakoutPrice = shortBreakoutPrice
      if direction == None:
        continue
      self.generateTrade(symbol, direction, breakoutPrice, cmp)
  def generateTrade(self, tradingSymbol, direction, breakoutPrice, cmp):
    trade = Trade(tradingSymbol)
    trade.strategy = self.getName()
    trade.direction = direction
    trade.productType = self.productType
    trade.placeMarketOrder = True
    trade.requestedEntry = breakoutPrice
    trade.timestamp = Utils.getEpoch(self.startTimestamp) 
    trade.qty = int(self.calculateCapitalPerTrade() / breakoutPrice)
    if trade.qty == 0:
      trade.qty = 1 
    if direction == 'LONG':
```

SampleStrategy_part1 is a concrete, example trading strategy that plugs into the framework by subclassing BaseStrategy and demonstrating the full signal-to-trade setup for simple intraday equity breakouts. It enforces a singleton pattern via an __instance holder and getInstance so only one SAMPLE strategy instance runs, then calls BaseStrategy initialization with the strategy name to inherit common wiring (including TradeManager registration and preloaded trades). During initialization it configures strategy-level parameters: it chooses MIS as the productType, lists a small set of cash symbols to scan, sets a small SL and target percentage, sets start/stop/square-off timestamps using Utils.getTimeOfToDay (as provided by the shared Utils layer), and sets capital, leverage, maxTradesPerDay and isFnO to reflect a simple intraday equities approach. The process method is the signal generator: it first enforces the daily trade cap by comparing the current trades list (populated by BaseStrategy via TradeManager) to maxTradesPerDay, then iterates the configured symbols. For each symbol it fetches a normalized quote via getQuote (which ultimately uses Quotes.getQuote), logs and skips if no quote is available, computes a tiny breakout band around the quote close by applying a 0.5% offset and normalizing with Utils.roundToNSEPrice, reads the last traded price, and decides direction = LONG if the CMP exceeds the long breakout level or SHORT if it falls below the short breakout level. When a breakout is detected it calls generateTrade with the symbol, direction, breakout price and CMP. generateTrade builds a Trade object, tags it with the strategy name and direction, sets product type, marks it as a market-placement candidate, stamps the trade timestamp using Utils.getEpoch on the strategy start timestamp and sizes the order by dividing calculateCapitalPerTrade (inherited from BaseStrategy) by the breakout price, ensuring at least one share. The rest of trade finalization and lifecycle (validation via shouldPlaceTrade, order creation and handoff to TradeManager/OrderManager) is implemented in the paired SampleStrategy_part2 and base trade lifecycle pieces; SampleStrategy_part1’s role is therefore to produce clean, sized entry signals for the engine to validate and execute. This pattern mirrors the other example strategies such as BNFORB30Min

```python
# file path: src/strategies/SampleStrategy.py
      trade.stopLoss = Utils.roundToNSEPrice(breakoutPrice - breakoutPrice * self.slPercentage / 100)
      if cmp < trade.stopLoss:
        trade.stopLoss = Utils.roundToNSEPrice(cmp - cmp * 1 / 100)
    else:
      trade.stopLoss = Utils.roundToNSEPrice(breakoutPrice + breakoutPrice * self.slPercentage / 100)
      if cmp > trade.stopLoss:
        trade.stopLoss = Utils.roundToNSEPrice(cmp + cmp * 1 / 100)
    if direction == 'LONG':
      trade.target = Utils.roundToNSEPrice(breakoutPrice + breakoutPrice * self.targetPercentage / 100)
    else:
      trade.target = Utils.roundToNSEPrice(breakoutPrice - breakoutPrice * self.targetPercentage / 100)
    trade.intradaySquareOffTimestamp = Utils.getEpoch(self.squareOffTimestamp)
    TradeManager.addNewTrade(trade)
  def shouldPlaceTrade(self, trade, tick):
    if super().shouldPlaceTrade(trade, tick) == False:
      return False
    if tick == None:
      return False
    if trade.direction == Direction.LONG and tick.lastTradedPrice > trade.requestedEntry:
      return True
    elif trade.direction == Direction.SHORT and tick.lastTradedPrice < trade.requestedEntry:
      return True
    return False
```

SampleStrategy_part2 is the execution-side implementation that takes a breakout signal produced by SampleStrategy_part1, converts it into a fully-specified Trade object with risk parameters, and hands that Trade to the framework’s lifecycle manager so it can be placed and tracked. When preparing the trade it first computes a directional stop loss by anchoring to the breakoutPrice and applying the strategy’s slPercentage, using Utils.roundToNSEPrice to normalize the price to exchange granularity; it then applies a defensive adjustment based on the current market price (cmp) so the stop does not end up on the wrong side of the market, again rounded via Utils.roundToNSEPrice. The target is derived similarly from breakoutPrice using the strategy’s targetPercentage and rounded. The method also stamps the trade with an intraday square-off epoch using Utils.getEpoch on the strategy’s squareOffTimestamp so the engine can automatically exit at session end, and finally registers the trade with TradeManager.addNewTrade so the order manager and TradeManager lifecycle routines will pick it up for placement, SL/target tracking and persistence. The shouldPlaceTrade override first delegates to BaseStrategy.shouldPlaceTrade so the common gating logic (market window, per-strategy limits, etc.) runs; it then requires a non-null market tick and enforces a simple execution trigger: for a LONG the tick lastTradedPrice must have moved above the requestedEntry, and for a SHORT it must have moved below the requestedEntry. This placement gating mirrors the pattern used in BNFORB30Min_part2 and ensures the strategy only converts queued trades into live orders when the market confirms the breakout.

```python
# file path: src/strategies/ShortStraddleBNF.py
import logging
from datetime import datetime
from instruments.Instruments import Instruments
from models.Direction import Direction
from models.ProductType import ProductType
from strategies.BaseStrategy import BaseStrategy
from utils.Utils import Utils
from trademgmt.Trade import Trade
from trademgmt.TradeManager import TradeManager
```

For ShortStraddleBNF these imports connect the strategy to the engine layers it needs: logging provides the runtime diagnostics and event traces the strategy will emit; datetime supplies the timestamping and time-based decisions such as marking entries, exits and comparing against option expiries; Instruments gives access to instrument metadata and lookup utilities so the strategy can translate conceptual strikes and expiries into concrete exchange symbols (critical for option mechanics); Direction and ProductType bring in the domain enums the strategy uses when it constructs Trade intent (side and product routing); BaseStrategy is the superclass that wires the strategy into the framework’s lifecycle and scheduling (the same inheritance pattern you saw in BNFORB30Min_part1); Utils supplies shared helper routines (recall the two small option-facing helpers we covered in Utils_part4 live alongside this and the strategy will call the broader Utils for ancillary tasks); Trade is the domain object used to represent a planned execution with all its parameters; and TradeManager is the execution/lifecycle handoff that accepts Trade instances and runs order placement, monitoring and exit logic (the piece StartAlgoAPI ultimately causes to run once the algo is started). This grouping follows the common import pattern across strategies—pulling in models, instruments, utility and trade management—but differs from some other strategies that also import real-time Quotes or explicit timing helpers; the presence of Instruments plus Trade/TradeManager here signals that ShortStraddleBNF focuses on option symbol construction and full trade lifecycle handling.

```python
# file path: src/strategies/ShortStraddleBNF.py
class ShortStraddleBNF(BaseStrategy):
  __instance = None
  @staticmethod
  def getInstance(): 
    if ShortStraddleBNF.__instance == None:
      ShortStraddleBNF()
    return ShortStraddleBNF.__instance
  def __init__(self):
    if ShortStraddleBNF.__instance != None:
      raise Exception("This class is a singleton!")
    else:
      ShortStraddleBNF.__instance = self
    super().__init__("ShortStraddleBNF")
    self.productType = ProductType.MIS
    self.symbols = []
    self.slPercentage = 30
    self.targetPercentage = 0
    self.startTimestamp = Utils.getTimeOfToDay(11, 0, 0) 
    self.stopTimestamp = Utils.getTimeOfToDay(14, 0, 0) 
    self.squareOffTimestamp = Utils.getTimeOfToDay(14, 30, 0) 
    self.capital = 100000 
    self.leverage = 0
    self.maxTradesPerDay = 2 
    self.isFnO = True 
    self.capitalPerSet = 100000 
  def canTradeToday(self):
    return True
  def process(self):
    now = datetime.now()
    if now < self.startTimestamp:
      return
    if len(self.trades) >= self.maxTradesPerDay:
      return
    futureSymbol = Utils.prepareMonthlyExpiryFuturesSymbol('BANKNIFTY')
    quote = self.getQuote(futureSymbol)
    if quote == None:
      logging.error('%s: Could not get quote for %s', self.getName(), futureSymbol)
      return
    ATMStrike = Utils.getNearestStrikePrice(quote.lastTradedPrice, 100)
    logging.info('%s: Nifty CMP = %f, ATMStrike = %d', self.getName(), quote.lastTradedPrice, ATMStrike)
    ATMCESymbol = Utils.prepareWeeklyOptionsSymbol("BANKNIFTY", ATMStrike, 'CE')
    ATMPESymbol = Utils.prepareWeeklyOptionsSymbol("BANKNIFTY", ATMStrike, 'PE')
    logging.info('%s: ATMCESymbol = %s, ATMPESymbol = %s', self.getName(), ATMCESymbol, ATMPESymbol)
    self.generateTrades(ATMCESymbol, ATMPESymbol)
  def generateTrades(self, ATMCESymbol, ATMPESymbol):
    numLots = self.calculateLotsPerTrade()
    quoteATMCESymbol = self.getQuote(ATMCESymbol)
    quoteATMPESymbol = self.getQuote(ATMPESymbol)
    if quoteATMCESymbol == None or quoteATMPESymbol == None:
      logging.error('%s: Could not get quotes for option symbols', self.getName())
      return
    self.generateTrade(ATMCESymbol, numLots, quoteATMCESymbol.lastTradedPrice)
    self.generateTrade(ATMPESymbol, numLots, quoteATMPESymbol.lastTradedPrice)
    logging.info('%s: Trades generated.', self.getName())
  def generateTrade(self, optionSymbol, numLots, lastTradedPrice):
    trade = Trade(optionSymbol)
    trade.strategy = self.getName()
    trade.isOptions = True
    trade.direction = Direction.SHORT 
    trade.productType = self.productType
```

ShortStraddleBNF_part1 is a singleton subclass of BaseStrategy that implements the signal-generation and wiring for a short straddle on Bank Nifty: only one instance can exist (same pattern used by BNFORB30Min and OptionSelling) and the constructor overrides BaseStrategy defaults to set strategy-specific parameters such as productType, slPercentage, targetPercentage, start/stop/square-off timestamps (using Utils.getTimeOfToDay), capital, leverage, maxTradesPerDay, and isFnO. canTradeToday always returns true for this strategy, so the decision to act is driven by runtime checks and limits. The process method runs sequentially: it first enforces the start time and daily trade-count limit, then builds the Bank Nifty monthly-futures symbol using the shared Utils helper and fetches its quote via BaseStrategy.getQuote. If the futures quote is available, it computes the ATM strike by rounding the futures last traded price to the nearest configured strike interval using the nearest-strike helper from Utils_part4 (this strategy uses a 100-point granularity), logs the computed ATM, then builds the corresponding weekly option symbols for both CE and PE using the weekly-options helper. generateTrades computes how many lots to trade using the BaseStrategy lot calculation, fetches quotes for both option symbols, aborts with a log if either option quote is missing, and otherwise invokes generateTrade for each side. generateTrade constructs a Trade object for the option, tags it with the strategy name, marks it as

```python
# file path: src/strategies/ShortStraddleBNF.py
    trade.placeMarketOrder = True
    trade.requestedEntry = lastTradedPrice
    trade.timestamp = Utils.getEpoch(self.startTimestamp) 
    isd = Instruments.getInstrumentDataBySymbol(optionSymbol) 
    trade.qty = isd['lot_size'] * numLots
    trade.stopLoss = Utils.roundToNSEPrice(trade.requestedEntry + trade.requestedEntry * self.slPercentage / 100)
    trade.target = 0 
    trade.intradaySquareOffTimestamp = Utils.getEpoch(self.squareOffTimestamp)
    TradeManager.addNewTrade(trade)
  def shouldPlaceTrade(self, trade, tick):
    if super().shouldPlaceTrade(trade, tick) == False:
      return False
    return True
  def getTrailingSL(self, trade):
    if trade == None:
      return 0
    if trade.entry == 0:
      return 0
    lastTradedPrice = TradeManager.getLastTradedPrice(trade.tradingSymbol)
    if lastTradedPrice == 0:
      return 0
    trailSL = 0
    profitPoints = int(trade.entry - lastTradedPrice)
    if profitPoints >= 5:
      factor = int(profitPoints / 5)
      trailSL = Utils.roundToNSEPrice(trade.initialStopLoss - factor * 5)
    logging.info('%s: %s Returning trail SL %f', self.getName(), trade.tradingSymbol, trailSL)
    return trailSL
```

ShortStraddleBNF_part2 is the execution-facing half that ties the ShortStraddleBNF decision logic into the instrument metadata layer and the TradeManager lifecycle, so when ShortStraddleBNF_part1 decides to create positions the trades are constructed, sized, timestamped and handed off for execution and tracking. When a short option trade is generated here, a Trade object is populated as an options, short-direction market entry: it sets the market-order flag and records the requested entry as the last traded price, stamps the trade time using Utils.getEpoch on the strategy start time, looks up instrument metadata via Instruments.getInstrumentDataBySymbol to compute the quantity from the lot size and requested number of lots, and calculates the static stop-loss by applying the strategy's slPercentage to the requested entry and normalizing the result with Utils.roundToNSEPrice. The trade target is left at zero and the intraday square-off time is derived from the strategy's squareOffTimestamp; finally the populated trade is handed to TradeManager.addNewTrade so it is persisted and enters the framework’s order and lifecycle machinery (note Instruments and TradeManager calls produce console/file side effects as part of that handoff). The shouldPlaceTrade implementation in ShortStraddleBNF_part2 simply defers to BaseStrategy.shouldPlaceTrade, so the strategy relies on the framework’s generic gating (time windows, enabled flag, max trades per day, etc.) without adding extra entry conditions. The getTrailingSL override implements a point-based trailing stop for short option positions: it guards against missing trade or price data, fetches the current last traded price via TradeManager.get

```python
# file path: src/restapis/BrokerLoginAPI.py
import logging
from flask.views import MethodView
from flask import request, redirect
from core.Controller import Controller
```

These imports load the small set of runtime pieces BrokerLoginAPI needs to act as the HTTP surface for broker authentication: Python's logging so handlers in this module can emit diagnostic and audit messages consistent with the rest of the app; Flask's MethodView so BrokerLoginAPI can be implemented as a class-based endpoint like other web-facing controls in the project; Flask's request and redirect helpers so the endpoint can read incoming HTTP data (query parameters, posted form data, callbacks) and issue browser redirects during OAuth-style flows; and the core Controller so the HTTP layer can delegate broker-specific authentication logic, token exchange and session setup to the centralized controller. Compared to other import groups in the codebase that also bring in JSON utilities, configuration loaders, broker model classes or concrete broker adapters such as ZerodhaLogin, this module deliberately keeps its imports minimal and high-level — it does not pull in broker implementation details itself but relies on Controller to encapsulate those concerns, mirroring the same separation of HTTP surface and business logic you saw with StartAlgoAPI.

```python
# file path: src/restapis/BrokerLoginAPI.py
class BrokerLoginAPI(MethodView):
  def get(self):
    redirectUrl = Controller.handleBrokerLogin(request.args)
    return redirect(redirectUrl, code=302)
```

BrokerLoginAPI implements the web-facing GET endpoint that starts and completes the broker authentication handshake for the framework. When its get handler is invoked it forwards the incoming query parameters to Controller.handleBrokerLogin; Controller then reads the broker app configuration, constructs a BrokerAppDetails instance, chooses the provider-specific login adapter (for example ZerodhaLogin), and either returns a broker login URL for the browser to be sent to or performs the request_token → access_token exchange and sets the Controller's brokerLogin handle. BrokerLoginAPI then issues an HTTP redirect response to whatever URL Controller.handleBrokerLogin returned (typically the broker's login page or the framework home URL with a logged-in flag). This keeps all broker-specific auth mechanics in Controller and the provider adapters while providing a simple HTTP entry point the UI can call; other components like BaseOrderManager, BaseTicker, HoldingsAPI and PositionsAPI obtain the authenticated broker handle later via Controller.getBrokerLogin and therefore depend on BrokerLoginAPI having completed this flow.

```python
# file path: src/restapis/HoldingsAPI.py
from flask.views import MethodView
import json
import logging
from core.Controller import Controller
```

HoldingsAPI imports MethodView from Flask so the endpoint is implemented as a class-based view, keeping its HTTP surface consistent with other web-facing controls in the framework. The standard json module is brought in to serialize and deserialize the holdings payloads exchanged over the REST surface, allowing the endpoint to return structured portfolio data to clients. logging is used so the API can emit the same runtime diagnostics and audit-style messages the rest of the app produces when handling requests and reporting errors. Controller from core.Controller is imported so the web layer can delegate the actual business work—auth-aware session lookup, broker adaptor interaction, and holdings normalization—to the centralized controller that knows how to talk to broker backends and map their data into the framework’s domain objects. This combination mirrors the pattern used by BrokerLoginAPI (which also relied on MethodView, logging and Controller) but differs in that HoldingsAPI focuses on returning structured data and therefore does not pull in request/redirect helpers used during OAuth-style flows.

```python
# file path: src/restapis/HoldingsAPI.py
class HoldingsAPI(MethodView):
  def get(self):
    brokerHandle = Controller.getBrokerLogin().getBrokerHandle()
    holdings = brokerHandle.holdings()
    logging.info('User holdings => %s', holdings)
    return json.dumps(holdings)
```

When an HTTP GET arrives at the holdings endpoint, HoldingsAPI.get runs and drives a small, linear flow that bridges the web layer to the broker session already established by the framework. It first asks Controller.getBrokerLogin for the current broker login instance so it reuses the authenticated session that was created earlier by Controller.handleBrokerLogin (the flow BrokerLoginAPI triggers during user authentication). From that BaseLogin-derived object it retrieves the broker handle via getBrokerHandle; the broker handle is the runtime adaptor that speaks to the brokerage and holds the active credentials/session context that BaseLogin stored during login. HoldingsAPI then calls the broker handle's holdings method, which performs the network request to the broker to fetch the user's portfolio; the returned holdings are logged with logging.info and serialized to JSON for the HTTP response. This follows the same pattern you saw in PositionsAPI (the only functional difference is calling holdings instead of positions) and exists to provide the dashboard and other components a simple, authenticated REST surface for real-time portfolio data.

```python
# file path: src/restapis/PositionsAPI.py
from flask.views import MethodView
import json
import logging
from core.Controller import Controller
```

These imports bring together the minimal pieces PositionsAPI needs to present positions over HTTP and hand off the real work to the controller: MethodView from flask.views provides the class-based endpoint pattern used throughout the web layer so PositionsAPI can be implemented like the other API classes; the json module is used to serialize and deserialize the position payloads the endpoint returns (PositionsAPI returns structured JSON rather than rendering HTML templates); logging supplies the same diagnostic and audit emission capability you saw used in BrokerLoginAPI so runtime events and errors from this endpoint are recorded consistently with the rest of the app; and Controller from core.Controller is the centralized business-logic entry point that PositionsAPI delegates to for fetching, normalizing and preparing current position data before returning it to authenticated clients. This set mirrors the common pattern used by other endpoints that combine MethodView and Controller, but it omits template or redirect helpers because its purpose is to return machine-readable position data.

```python
# file path: src/restapis/PositionsAPI.py
class PositionsAPI(MethodView):
  def get(self):
    brokerHandle = Controller.getBrokerLogin().getBrokerHandle()
    positions = brokerHandle.positions()
    logging.info('User positions => %s', positions)
    return json.dumps(positions)
```

PositionsAPI implements the HTTP surface for position data: when its GET handler is invoked by an authenticated client it obtains the active broker session from Controller.getBrokerLogin and then asks that broker handle for the current positions, which triggers a network call into the broker adaptor to retrieve live position information. Because Controller.brokerLogin is created and populated during the BrokerLoginAPI handshake, PositionsAPI relies on that BaseLogin-backed session being present to reach the broker; the broker handle it receives is the broker-specific object that encapsulates the API details and returns positions in a normalized form for the framework. After fetching positions, the GET handler records a runtime info-level entry and returns the positions serialized as JSON to the caller. This pattern mirrors HoldingsAPI: both endpoints are simple MethodView entry points that delegate to the Controller-backed broker handle to perform the actual broker I/O and then log and return the resulting JSON.

```python
# file path: src/Test.py
import logging
import time
from core.Controller import Controller
from ticker.ZerodhaTicker import ZerodhaTicker
from ordermgmt.ZerodhaOrderManager import ZerodhaOrderManager
from ordermgmt.Order import Order
from core.Quotes import Quotes
from utils.Utils import Utils
```

Test.py pulls together the runtime helpers and the Zerodha-specific adapters so the harness can bring up a live feed, exercise order paths and observe normalized quote data end-to-end. It uses the logging module so the harness emits the same diagnostic traces and runtime messages the rest of the app relies on for debugging and audits, and it uses the time module to pace the smoke-test loop and insert short sleeps or timeouts while the ticker and order manager are being exercised. Controller is imported so the test can reuse the centralized broker/session orchestration and credential context (the same coordination point used by BrokerLoginAPI). ZerodhaTicker is the real-time feed implementation the test instantiates to validate connectivity and tick delivery, while ZerodhaOrderManager and Order let the harness create and submit representative orders against the Zerodha execution adaptor and observe the lifecycle. Quotes is pulled in so the test can route raw ticks through the same quote normalization layer the engine expects, and Utils supplies small shared helpers (timestamping, formatting, and other utilitarian routines) used by the test flow. This set of imports follows the project’s usual pattern of combining logging and Utils for common functionality, but differs from the other import lists you’ve seen by intentionally wiring in Zerodha-specific classes (ZerodhaTicker and ZerodhaOrderManager) instead of the generic BaseOrderManager or the kiteconnect client used elsewhere; the addition of time is also specific to a test harness rather than production endpoints.

```python
# file path: src/Test.py
class Test:
  def testTicker():
    ticker = ZerodhaTicker()
    ticker.startTicker()
    ticker.registerListener(Test.tickerListener)
    time.sleep(5)
    ticker.registerSymbols(['SBIN', 'RELIANCE'])
    time.sleep(60)
    logging.info('Going to stop ticker')
    ticker.stopTicker()
  def tickerListener(tick):
    logging.info('tickerLister: onNewTick %s', vars(tick));
  def testOrders():
    orderManager = ZerodhaOrderManager()
    exchange = 'NSE';
    tradingSymbol = 'SBIN'
    lastTradedPrice = Quotes.getCMP(exchange + ':' + tradingSymbol)
    logging.info(tradingSymbol + ' CMP = %f', lastTradedPrice)
    limitPrice = lastTradedPrice - lastTradedPrice * 1 / 100
    limitPrice = Utils.roundToNSEPrice(limitPrice)
    qty = 1
    direction = 'LONG'
    origOrderId = orderManager.placeOrder(tradingSymbol, limitPrice, qty, direction)
    logging.info('Original order Id %s', origOrderId)
    time.sleep(10)
    newPrice = lastTradedPrice
    if origOrderId:
        orderManager.modifyOrder(origOrderId, newPrice)
    time.sleep(10)
    slPrice = newPrice - newPrice * 1 / 100
    slPrice = Utils.roundToNSEPrice(slPrice)
    slDirection = 'SHORT' if direction == 'LONG' else 'LONG'
    slOrderId = orderManager.placeSLOrder(tradingSymbol, slPrice, qty, slDirection)
    logging.info('SL order Id %s', slOrderId)
    time.sleep(10)
    targetPrice = newPrice + newPrice * 2 / 100
    targetPrice = Utils.roundToNSEPrice(targetPrice)
    targetDirection = 'SHORT' if direction == 'LONG' else 'LONG'
    targetOrderId = orderManager.placeOrder(tradingSymbol, targetPrice, qty, targetDirection)
    logging.info('Target order Id %s', targetOrderId)
    time.sleep(10)
    if targetOrderId:
        orderManager.cancelOrder(targetOrderId)
        logging.info('Cancelled Target order Id %s', targetOrderId)
    logging.info("Algo done executing all orders. Check ur orders and positions in broker terminal.") 
  def testMisc():
    orderManager = ZerodhaOrderManager()
    sampleOrder = Order(orderInputParams=None)
    sampleOrder.orderId='210505200078243'
    orders = []
    orders.append(sampleOrder)
    orderManager.fetchAndUpdateAllOrderDetails(orders)
```

Test is a small harness that exercises the ticker and order-management layers so you can do a quick smoke test of broker connectivity, real-time feed handling and basic order workflows. testTicker instantiates ZerodhaTicker and starts its connection path (recall ZerodhaTicker.startTicker wires up the broker ticker callbacks and connects the KiteTicker as you saw earlier), then registers the Test.tickerListener via BaseTicker.registerListener so incoming ticks will be dispatched to the listener; it then pauses briefly to let the socket settle, asks the ticker to subscribe to two sample symbols (SBIN and RELIANCE) which triggers the instrument-resolution logic you saw in Instruments.getInstrumentDataBySymbol and ZerodhaTicker.registerSymbols, waits a minute to collect live ticks and finally stops the ticker to close the connection. tickerListener is a simple callback that receives tick objects from the feed and writes their attributes to the log so you can confirm the on_ticks → TickData conversion and listener dispatch path is working. testOrders drives the order manager path: it creates a ZerodhaOrderManager, reads a current market price using Quotes.getCMP, computes a rounded limit price using Utils.roundToNSEPrice, and places an entry order through ZerodhaOrderManager.placeOrder; after a pause it attempts a modify via ZerodhaOrderManager.modifyOrder, then it constructs and places an SL order and a target order (using the direction inversion logic you expect for protective and profit exits), logs the returned IDs, waits, cancels the target order if present, and then logs completion — exercising place, modify, place-SL, place-target and cancel flows so you can validate BaseOrderManager and ZerodhaOrderManager interactions with the broker. testMisc exercises the order-detail retrieval flow by creating a sample Order, seeding it with a real-looking orderId, and calling ZerodhaOrderManager.fetchAndUpdateAllOrderDetails so the manager will query the broker for latest order state and update the Order object; this validates the fetch-and-sync logic that TradeManager and other lifecycle components rely on. Overall, Test ties together ZerodhaTicker, BaseTicker listener dispatch, Instruments resolution, Quotes and Utils helpers, and ZerodhaOrderManager order APIs to give a compact end-to-end sanity check of the ticker and order layers you already inspected.

```python
# file path: src/core/Algo.py
import logging
import threading
import time
from instruments.Instruments import Instruments
from trademgmt.TradeManager import TradeManager
from strategies.SampleStrategy import SampleStrategy
from strategies.BNFORB30Min import BNFORB30Min
from strategies.OptionSelling import OptionSelling
from strategies.ShortStraddleBNF import ShortStraddleBNF
```

The imports assemble the pieces this orchestration entrypoint needs to bridge instrument setup and strategy execution within the sdoosa-algo-trade-python-master_cleaned framework. logging provides the same runtime diagnostics and event traces you saw earlier. threading and time supply simple concurrency and scheduling primitives the entrypoint uses to run strategy loops or idle waits without blocking the process. Instruments gives access to the instrument metadata layer so the entrypoint can load exchange symbols and lookups before any strategy runs (recall Instruments.fetchInstruments from the architecture description). TradeManager brings in the execution and lifecycle handoff so once a strategy decides on trades those TradeManager capabilities are available to create, monitor and manage orders. Finally, the concrete strategy classes SampleStrategy, BNFORB30Min, OptionSelling and ShortStraddleBNF are imported so the entrypoint can choose which strategy to instantiate and run; ShortStraddleBNF is the strategy this file wires up for a short straddle on Bank Nifty while the others are available as alternate or sample strategies. This pattern mirrors other modules that import Instruments and TradeManager, but differs by pulling in threading/time for runtime orchestration and by importing multiple concrete strategies here rather than only base strategy abstractions.

```python
# file path: src/core/Algo.py
class Algo:
  isAlgoRunning = None
  @staticmethod
  def startAlgo():
    if Algo.isAlgoRunning == True:
      logging.info("Algo has already started..")
      return
    logging.info("Starting Algo...")
    Instruments.fetchInstruments()
    tm = threading.Thread(target=TradeManager.run)
    tm.start()
    time.sleep(2)
    threading.Thread(target=ShortStraddleBNF.getInstance().run).start()
    Algo.isAlgoRunning = True
    logging.info("Algo started.")
```

Algo acts as the orchestration entrypoint that boots the runtime pieces needed for a trading run. Its startAlgo method first uses a static isAlgoRunning flag to prevent duplicate starts and logs the activity; it then triggers Instruments.fetchInstruments so the strategy layer has the instrument metadata required to resolve strikes, expiries and exchange symbols. After that it launches TradeManager.run in a background thread so the market-data ticker, trade persistence and order-tracking loop are initialized (TradeManager.run is responsible for starting the ticker, loading prior trades, waiting for market open and continuously managing orders). startAlgo then pauses briefly to give TradeManager time to initialize and then starts the ShortStraddleBNF singleton’s run method in its own thread; ShortStraddleBNF.run follows the BaseStrategy lifecycle you’ve already seen (checking enabled state, market timings, canTradeToday, and then generating Trade instances that are handed off to TradeManager). Finally startAlgo marks Algo.isAlgoRunning as true and logs that the algo has started. The ordering ensures instrument metadata and the TradeManager’s market and execution plumbing are ready before the strategy begins producing trade intents.

```python
# file path: src/ticker/ZerodhaTicker.py
  def on_connect(self, ws, response):
    self.onConnect()
  def on_close(self, ws, code, reason):
    self.onDisconnect(code, reason)
  def on_error(self, ws, code, reason):
    self.onError(code, reason)
  def on_reconnect(self, ws, attemptsCount):
    self.onReconnect(attemptsCount)
  def on_noreconnect(self, ws):
    self.onMaxReconnectsAttempt()
  def on_order_update(self, ws, data):
    self.onOrderUpdate(data)
```

ZerodhaTicker_part2 provides the thin adapter layer that turns KiteTicker websocket callbacks into the framework’s unified BaseTicker lifecycle hooks so the rest of the engine can react in a broker-agnostic way. When the KiteTicker signals a successful connection, the on_connect implementation forwards that event into BaseTicker.onConnect so the common connection-handling logic and any registered listeners are notified. When the socket closes, the on_close adapter sends the close code and reason into BaseTicker.onDisconnect; when an error is reported, on_error forwards the error details into BaseTicker.onError. Reconnect progress and terminal reconnect failure are handled similarly: on_reconnect relays the attempt count into BaseTicker.onReconnect, and on_noreconnect invokes BaseTicker.onMaxReconnectsAttempt to indicate the ticker has exhausted automatic retries. Finally, broker order lifecycle messages from KiteTicker arrive at on_order_update and are passed unchanged into BaseTicker.onOrderUpdate so order managers or higher-level components can process fills, modifications or cancels. These methods are the callbacks ZerodhaTicker_part1 wires onto the KiteTicker before starting the connection, allowing the Project’s TradeManager, tests and listeners to see a consistent stream of connection and order events.

```python
# file path: config/brokerapp.json
{
  "broker": "zerodha",
  "clientID": "dummy",
  "appKey": "dummy",
  "appSecret": "dummy",
  "redirectUrl": "http://localhost:8080/apis/broker/login/zerodha"
}
```

brokerapp.json is the single-source configuration the broker connectivity and authentication layers read to know which broker adaptor to use and how to authenticate; it declares the broker identifier and the three credential-ish fields clientID, appKey and appSecret that Controller.handleBrokerLogin reads via getBrokerAppConfig and copies into a BrokerAppDetails instance so the Controller can instantiate the correct broker login adapter (for example ZerodhaLogin when the broker value is zerodha). The file also contains redirectUrl which is the callback URL the broker’s OAuth-style handshake will return to and which ties directly into the web-facing BrokerLoginAPI endpoint flow: BrokerLoginAPI initiates the login, Controller delegates to the broker-specific login class using the BrokerAppDetails values, and the redirectUrl is used in that handshake so the broker returns control to the framework. In the repository snapshot you inspected, the credential fields are set to dummy placeholders, signaling that these values are intended to be replaced with real client/app credentials and a reachable callback URL for an actual authentication run.

```python
# file path: config/holidays.json
[
  "2021-01-26",
  "2021-03-11",
  "2021-03-29",
  "2021-04-02",
  "2021-04-14",
  "2021-04-21",
  "2021-05-13",
  "2021-07-21",
  "2021-08-19",
  "2021-09-10",
  "2021-10-15",
  "2021-11-04",
  "2021-11-05",
  "2021-11-19"
]
```

holidays.json is the canonical, standalone list of market-closure dates the trading engine uses to know when the exchange is closed and normal trading activity must be suppressed. It is represented as a simple JSON array of ISO date strings for 2021 that correspond to national/exchange holidays (for example Republic Day, Holi, Good Friday and other closures) and is intended to be read by the framework before runtime operations begin. At startup or when scheduling trading activity the framework calls getHolidays to parse this file and return the list; the scheduler, risk controls and the trade lifecycle manager consult that list to stop market-data ingestion, prevent order placement and to adjust session handling on days the market is closed. Unlike server.json and system.json, which are keyed objects containing a few runtime settings, holidays.json is a flat array of dates; it can also be extended to include per-day session metadata if needed, but the current file contains only date entries. Because holidays.json is standalone, it acts as a single source of truth for trading-disabled days that the rest of the components (Algo, the ticker adapters such as ZerodhaTicker_part2 via the scheduling/risk layer, and order-management routines) look up to enforce non-trading behavior.

```python
# file path: config/server.json
{
  "port": 8080,
  "enableSSL": false,
  "sslPort": 8443,
  "deployDir": "D:/temp/python-deploy",
  "logFileDir": "D:/temp/python-deploy/logs"
}
```

server.json centralizes the runtime server settings the framework uses when Algo boots the HTTP and WebSocket endpoints, providing the values that tell the process which network port to bind, whether TLS should be enabled and which TLS port to use, and where on disk the runtime should place deployed artifacts and log files. At startup the configuration loader function getServerConfig reads server.json and returns the parsed configuration so Algo and the server initialization path can consult the port and the enableSSL flag to decide whether to start plain HTTP or an SSL listener on the sslPort; other runtime components then consume deployDir to know the target directory for deployment artifacts (the same value later assigned to the deployDir variable in the codebase) and logFileDir as the location for runtime logs. Compared with system.json, which only supplies a higher-level homeUrl used by UI/navigation logic, server.json contains the concrete runtime and filesystem parameters the process needs to configure networking and local services.

```python
# file path: config/system.json
{
  "homeUrl": "http://localhost:8080"
}
```

system.json is the single-source place where the runtime announces the system’s public base URL to the rest of the framework; it contains a homeUrl entry that components read at startup to know the canonical external address for building callbacks, redirects and links. The framework’s loader getSystemConfig opens system.json and returns that JSON so orchestration and broker-related pieces can concatenate paths against the declared homeUrl instead of hard-coding host/port. In the project’s configuration pattern, system.json sits alongside server.json and brokerapp.json but serves a different role: server.json declares how the local server binds and where logs are written, brokerapp.json holds broker identity and credential fields that Controller.handleBrokerLogin maps into a BrokerAppDetails instance (including the broker redirect URL), and system.json supplies the logical public URL that those redirect endpoints and external integrations should use. Because system.json only provides the homeUrl, its purpose is narrowly scoped: give a single, authoritative base URL that other modules use when constructing external-facing endpoints.

```python
# file path: src/models/Direction.py
class Direction:
  LONG = "LONG"
  SHORT = "SHORT"
```

Direction is the small domain model that gives the rest of the framework a single, canonical way to express trade side semantics — the two named values capture whether a position or order is intended to be a long or a short. Across the orchestration layers such as Algo, the strategy implementations, the order-management and execution code, and the trade lifecycle manager, components compare against Direction constants instead of using ad-hoc strings so decisions about entries, exits, hedges and risk controls remain consistent. Because Direction is standalone it has no runtime wiring like ZerodhaTicker_part2 or the broker configuration read from brokerapp.json, but it is referenced conceptually by the same pieces you exercised with Test when orders are created or evaluated. The pattern behind Direction is the same approach used by OrderType, ProductType and Segment: each class enumerates a small closed set of domain values as named constants so the rest of sdoosa-algo-trade-python-master_cleaned can be type-safe and less error-prone when encoding exchange and strategy semantics; Direction differs only in scope and cardinality, modeling the binary side of a trade rather than order kinds, product categories, or market segments.

```python
# file path: src/models/OrderStatus.py
class OrderStatus:
  OPEN = "OPEN"
  COMPLETE = "COMPLETE"
  OPEN_PENDING = "OPEN PENDING"
  VALIDATION_PENDING = "VALIDATION PENDING"
  PUT_ORDER_REQ_RECEIVED = "PUT ORDER REQ RECEIVED"
  TRIGGER_PENDING = "TRIGGER PENDING"
  REJECTED = "REJECTED"
  CANCELLED = "CANCELLED"
```

OrderStatus defines the canonical set of named states used across the order-management and execution layers to represent where an order is in its lifecycle — from the moment a placement request enters the system through validation and broker interaction to final outcomes like completion, cancellation, or rejection. It provides labels such as an initial request-arrived state, a validation-pending state, one or more pending/open states that reflect awaiting broker acknowledgement or trigger activation, a final complete state for fully filled orders, and terminal states for rejected or cancelled orders. Algo and the higher-level orchestration create Order objects that carry an orderStatus field whose value is drawn from OrderStatus; broker adaptors (the ones selected via brokerapp.json) and the ticker adapter ZerodhaTicker_part2 drive updates to that field as they report confirms, fills, triggers, or error responses; the Test harness observes those transitions during smoke runs. Compared with TradeState, OrderStatus is more granular and focused specifically on order-level workflow steps (validation and broker request/trigger phases) while TradeState expresses broader trade lifecycle phases like created, active, or completed; with OrderType describing how an order should be executed (limit, market, stop variants), OrderStatus describes what has happened to that order during execution and monitoring, enabling consistent logging, risk checks, and state-driven behavior across the engine.

```python
# file path: src/models/OrderType.py
class OrderType:
  LIMIT = "LIMIT"
  MARKET = "MARKET"
  SL_MARKET = "SL_MARKET"
  SL_LIMIT = "SL_LIMIT"
```

OrderType is the small, standalone enumeration that the rest of the framework uses to name and normalize the execution intent of an order so strategies, the order manager and broker adapters all speak the same language. When a strategy constructs an order intent during a trade decision, it picks one of the OrderType values to express whether it should execute immediately at market, wait for a limit price, or act as a stop-loss instruction where a trigger price leads to either a market or a limit execution. That chosen OrderType then travels with the order through the order-management and validation layers (which your Test harness exercises against the broker connectivity), and finally the broker adapter (the adapter selected via the brokerapp.json-driven configuration) maps the normalized OrderType into the broker-specific API fields Zerodha or another broker expects. OrderType follows the same pattern used by ProductType, OrderStatus and TradeExitReason: each is a compact class of string constants used for cross-layer normalization. It differs from ProductType and OrderStatus by representing execution semantics (how to fill an order) rather than product segment or lifecycle state, and it specifically distinguishes plain market/limit execution from stop-loss variants so the order manager and brokers can handle triggers and execution-price behavior correctly.

```python
# file path: src/models/ProductType.py
class ProductType:
  MIS = "MIS"
  NRML = "NRML"
  CNC = "CNC"
```

ProductType is a tiny, centralized class that enumerates the product categories the trading engine uses when constructing and validating orders — the intraday margin product, the normal overnight product, and the delivery/cash product. Because the framework separates broker adaptors from core logic, ProductType provides the canonical labels the order-management and trade-lifecycle layers attach to every order so broker-specific encoders (for example, the Zerodha adaptor the orchestrator wires up via brokerapp.json) can map them to the broker API’s product parameter and the risk/margin subsystem can apply the correct rules. When Algo or the Test harness asks the order-management layer to place a trade, the chosen ProductType influences margin calculation, whether the position may be carried overnight, and which exit/auto-square-off rules apply. ProductType follows the same simple constant-pattern used by OrderType, Direction, and Segment: each is a compact namespace of string values used across the codebase to avoid magic literals. It differs from OrderType in that it describes the order’s billing/margin semantics rather than its execution mechanism, and it complements Direction and Segment by providing the product-dimension that, together with side and segment, fully describes how an order should be routed and managed.

```python
# file path: src/models/Segment.py
class Segment:
  EQUITY = "EQUITY"
  FNO = "FNO"
  CURRENCY = "CURRENCY"
  COMMADITY = "COMMADITY"
```

Segment is a small, canonical model that enumerates the market/exchange segments the framework recognizes — EQUITY, FNO, CURRENCY, and COMMADITY — and it centralizes the single set of labels that instrument normalization, data ingestion, and order routing consult when they need to apply segment-specific behavior. When instruments are normalized or instrument metadata is looked up, Segment values are attached to the instrument so ticker adapters like ZerodhaTicker_part2 and the ingestion layer present quotes with a consistent segment identity; the execution and order-management code then uses the same Segment labels to decide routing, margin/product rules, and any validations the trade lifecycle manager must enforce. Segment follows the same constant-enum pattern used by OrderType, Direction, and ProductType — a compact set of string constants referenced across the codebase rather than instantiated objects — but it is orthogonal to those: OrderType and Direction describe how an order should behave, ProductType describes contract/execution semantics, and Segment describes where the instrument lives in the market structure, so the runtime composes Segment with ProductType and OrderType to drive segment-aware risk controls and routing without scattering literal strings through Algo, the test harness, or other modules.

```python
# file path: src/restapis/HomeAPI.py
from flask.views import MethodView
from flask import render_template, request
```

HomeAPI pulls in Flask's class-based view helper MethodView so the route handlers can be implemented as tidy, testable classes rather than plain functions, and it imports render_template and request so the home endpoints can either render a simple HTML landing page and health/status UI or inspect incoming HTTP details (query parameters, headers and form data) for lightweight diagnostic and status checks. Within the sdoosa-algo-trade-python-master_cleaned framework, that keeps the root interface purposely minimal and decoupled from the orchestration pieces—other modules that expose API endpoints often import MethodView plus JSON, logging and the core Controller to perform broker orchestration and return machine-readable payloads, or import request plus redirect when they implement flow control; HomeAPI instead uses render_template to serve a human-friendly landing view and request to read any caller-supplied metadata for health checks, following the same class-based view pattern but without pulling in Controller or heavy JSON handling.

```python
# file path: src/restapis/HomeAPI.py
class HomeAPI(MethodView):
  def get(self):
    if 'loggedIn' in request.args and request.args['loggedIn'] == 'true':
      return render_template('index_loggedin.html')
    elif 'algoStarted' in request.args and request.args['algoStarted'] == 'true':
      return render_template('index_algostarted.html')
    else:
      return render_template('index.html')
```

HomeAPI is the lightweight Flask view that serves the application's root landing pages and lets simple clients (the browser UI, monitoring probes or smoke tests) verify runtime state by returning one of three HTML pages depending on URL query flags. As a MethodView it implements a GET handler that checks request query parameters for a loggedIn indicator and an algoStarted indicator; when loggedIn is present and true it returns the index_loggedin.html view (the page that exposes the Start Algo button tied to StartAlgoAPI), when algoStarted is present and true it returns the index_algostarted.html view (the page that shows the algo is running and offers stop controls), and if neither flag is set it returns the default index.html landing page. The behavioral link with StartAlgoAPI is explicit: StartAlgoAPI launches Algo.startAlgo in a background thread and responds with a redirect URL that appends the algoStarted flag so HomeAPI will render the running-state page; similarly the loggedIn flag maps the post-login state the UI wants to present after broker authentication. HomeAPI is intentionally minimal and stateless—its job is purely presentation routing for basic status and UI entry rather than orchestration or broker connectivity, so it lives standalone as the simple entry point for human and automated checks of the service.

```python
# file path: src/templates/index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Algo Trading - Python</title>
  <style>
    .button {
      background-color: #53a5b0;
      border: none;
      padding: 15px 32px;
      text-align: center;
      text-decoration: none;
      display: inline-block;
      font-size: 16px;
      margin: 4px 2px;
      cursor: pointer;
    }
    .center {
      text-align: center;
    }
  </style>
  <script>
    function loginToBroker() {
      window.location.href =  "http://localhost:8080" + "/apis/broker/login/zerodha"
    }
  </script>
</head>
<body>
  <h1 style="color: blue" class="center">Welcome to algo trading in python</h1>
  <div class="center">
    <button class="button" onClick="loginToBroker()">
      Login to Broker
    </button>
  </div>
</body>
</html>

```

index.html is the minimal landing template the web UI serves when a user has not yet authenticated a broker session; its job within the sdoosa-algo-trade-python-master_cleaned project is to present a simple, friendly entry point that triggers the broker authentication flow so the rest of the trading dashboard and engine can be used. The document defines small, embedded style rules for a reusable button and a center alignment class, renders a blue heading that welcomes the user, and shows a single action button wired to a browser-side function named loginToBroker that navigates the browser to the local server’s Zerodha broker login endpoint so the backend’s BrokerLoginAPI can perform the authentication handshake. Since HomeAPI already handles which HTML landing page to return based on request flags and runtime state, index.html acts as the unauthenticated variant in that flow; the related templates index_loggedin.html and index_algostarted.html reuse the same visual styles but extend behavior — they load jQuery and perform AJAX POSTs to start and stop the algorithm engine (the startAlgo and stopAlgo client functions) and display running-state text instead of a login button — whereas index.html purposefully remains static and focused solely on initiating the login redirect. The file contains no dynamic asset injection or live market UI itself; it simply funnels the user into the broker adapter’s login route so the broader engine layers (broker-specific connectivity, order manager, strategy layer) can be accessed once authentication completes.

```python
# file path: src/templates/index_algostarted.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Algo Trading - Python</title>
  <style>
    .button {
      background-color: #53a5b0;
      border: none;
      padding: 15px 32px;
      text-align: center;
      text-decoration: none;
      display: inline-block;
      font-size: 16px;
      margin: 4px 2px;
      cursor: pointer;
    }
    .center {
      text-align: center;
    }
  </style>
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
  <script>
    function stopAlgo() {
      var url =  "http://localhost:8080" + "/apis/algo/stop"
      var data = {};
      $.post(url, {},  
        function(data, status, xhr) {
          alert('status: ' + status + ', data: ' + JSON.stringify(data));
        },
        'json'
      );
    }
  </script>
</head>
<body>
  <h1 style="color: blue" class="center">Welcome to algo trading in python</h1>
  <div class="center">
    <h4>Algo is running...</h4>
  </div>
</body>
</html>

```

index_algostarted.html is the simple frontend view the user sees after an algorithm has been started; it plays the role of a lightweight runtime dashboard page in the overall algo-trading UI, showing the algo's running status and offering a client-side hook to stop the strategy by calling the backend API. The template sets up basic HTML metadata and reuses the shared inline styles for the .button and .center classes used across the other entry pages (index_loggedin.html and index.html), then pulls in jQuery from a CDN so that the page can make AJAX calls. A small JavaScript routine named stopAlgo is defined that issues a POST to the engine's stop endpoint and reports the JSON response via a browser alert, so the page can command the engine to halt execution at runtime. The visible body renders a branded heading and a concise "Algo is running..." status message centered on the page, presenting minimal runtime feedback to the operator. Remember the HomeAPI we reviewed earlier that serves the app's landing pages; index_algostarted.html fits the same pattern as those templates—index_loggedin.html provides a startAlgo routine and a start button, and index.html provides a loginToBroker navigation—so index_algostarted.html is the counterpart used when the system is in the running state and needs a client-side stop control wired to the /apis/algo/stop endpoint.

```python
# file path: src/templates/index_loggedin.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Algo Trading - Python</title>
  <style>
    .button {
      background-color: #53a5b0;
      border: none;
      padding: 15px 32px;
      text-align: center;
      text-decoration: none;
      display: inline-block;
      font-size: 16px;
      margin: 4px 2px;
      cursor: pointer;
    }
    .center {
      text-align: center;
    }
  </style>
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
  <script>
    function startAlgo() {
      var url =  "http://localhost:8080" + "/apis/algo/start"
      var data = {};
      $.post(url, {},  
        function(data, status, xhr) {
          if (data.redirect) {
            window.location.href = data.redirect;
          } else {
            alert('status: ' + status + ', data: ' + JSON.stringify(data));
          }
        },
        'json'
      );
    }
  </script>
</head>
<body>
  <h1 style="color: blue" class="center">Welcome to algo trading in python</h1>
  <div class="center">
    <button class="button" onClick="startAlgo()">
      Start Algo
    </button>
  </div>
</body>
</html>

```

index_loggedin.html is the template HomeAPI returns for authenticated users and acts as the minimal main dashboard after login: it declares a small set of page-wide styles for a reusable button and centered layout, pulls in jQuery for AJAX, and defines a client-side startAlgo function that posts to the application's algo start API. When the server responds to the POST, the JavaScript looks for a redirect field in the JSON and navigates the browser there if present; if not, it presents a simple alert with the status and returned data. The body renders a centered welcome header and a single Start Algo button wired to startAlgo, so clicking it is the UI action that kicks off the strategy/execution lifecycle handled by the order manager and trade lifecycle manager on the backend. The template follows the same visual and scripting pattern as index.html and index_algostarted.html: index.html provides the broker login entry (loginToBroker), index_loggedin.html is the authenticated entry point that triggers algorithm startup, and index_algostarted.html represents the running state and offers a stop action (stopAlgo). The file is standalone, uses the same simple styling and client-side AJAX approach as its siblings, and serves as the browser-side trigger that transitions the system from an authenticated idle state into active algo execution.

```python
# file path: src/trademgmt/TradeEncoder.py
from json import JSONEncoder
```

The import brings in the JSONEncoder base class from Python’s standard json library so TradeEncoder can subclass it and provide a customized JSON serialization strategy for Trade objects used by the trade lifecycle manager. Since the file’s job is to convert internal Trade instances into portable representations for persistence and messaging, TradeEncoder derives from JSONEncoder to override the default handling of objects the standard serializer can’t natively encode (for example, datetimes, enum types like OrderType/ProductType, or nested domain objects). This is a narrower, intent-focused import compared with other modules that pull in the json module to call dumps/loads directly; here the framework needs the encoder base class so the custom encoder can be registered when the system serializes trades. A prior, simpler TradeEncoder in the project implemented default by returning an object’s attribute dictionary, and this import is what enables that pattern of providing a pluggable, class-based JSON encoding implementation.

```python
# file path: src/trademgmt/TradeEncoder.py
class TradeEncoder(JSONEncoder):
  def default(self, o):
    return o.__dict__
```

TradeEncoder is a tiny, focused serializer class that extends the standard JSONEncoder so the trade lifecycle manager and other modules can turn internal Trade instances into a portable dictionary/JSON form for persistence, messaging, or inter-module transport. Conceptually it overrides the encoder's default behavior to emit the Trade object's attribute map (i.e., the set of its public fields and their values) rather than trying to encode the object as an opaque type, which produces a flat mapping whose keys match the Trade class's fields such as tradeID, tradingSymbol, productType, entryOrder, slOrder, targetOrder, timestamps and P&L attributes. That dictionary shape directly complements the manual deserializer implemented in TradeManager_part7's convertJSONToTrade, which expects those specific field names to reconstruct a Trade and its nested Order objects; similarly, the serialized productType, orderType and segment-like fields align with the canonical enums we discussed earlier (OrderType, ProductType, Segment). The implementation is standalone and intentionally minimal: it leverages Python's JSONEncoder machinery (the same import used elsewhere) to provide a predictable, framework-wide way to serialize Trade instances into the portable format the rest of the system uses for storage and messaging.

```python
# file path: src/trademgmt/TradeExitReason.py
class TradeExitReason:
  SL_HIT = "SL HIT"
  TRAIL_SL_HIT = "TRAIL SL HIT"
  TARGET_HIT = "TARGET HIT"
  SQUARE_OFF = "SQUARE OFF"
  SL_CANCELLED = "SL CANCELLED"
  TARGET_CANCELLED = "TARGET CANCELLED"
```

TradeExitReason is a small canonical enumeration that names the standard, human-readable reasons why the trade lifecycle manager and related modules mark a trade as closed; it centralizes the labels used by the trade lifecycle manager, order/execution modules, risk controls, logging and analytics so everyone in the system reports the same exit semantics. Conceptually it contains constants for the typical exit events you care about in an algo-trading flow — a stop‑loss being hit, a trailing stop‑loss being hit, a profit target being hit, an explicit square‑off by the system or user, and the two cancellation cases where a stop or a target is cancelled — and those labels are what the rest of the engine stores and emits when a trade completes. The class follows the same lightweight pattern as TradeState and OrderType: a centralized set of string labels that other components import and compare against, but whereas TradeState captures where a trade is in its lifecycle, TradeExitReason captures why the lifecycle ended. In practice components like TradeManager use these constants when transitioning a trade to completed status or when logging order-management actions (for example TradeManager sets a trade to completed with one of these exit reasons and then calls its SL/target order helpers), and analytics and risk modules consume them to aggregate outcomes and enforce post‑exit rules. Because TradeExitReason is standalone and declarative, it keeps downstream code consistent and readable across the execution and lifecycle layers of the sdoosa-algo-trade-python-master_cleaned framework.

```python
# file path: src/trademgmt/TradeState.py
class TradeState:
  CREATED = 'created' 
  ACTIVE = 'active' 
  COMPLETED = 'completed' 
  CANCELLED = 'cancelled' 
  DISABLED = 'disabled'
```

TradeState defines the canonical set of high-level lifecycle labels the engine uses to track where a trade sits in its life so the trade lifecycle manager, order manager and persistence/reconciliation logic can reason about transitions consistently. When a Trade object is instantiated it starts with the CREATED label, which marks an in-memory record that has been constructed but not yet advanced; the engine and order manager move that trade to ACTIVE when entry execution completes and the position is live; when an exit completes successfully the trade is marked COMPLETED and the lifecycle manager finalizes timestamps, P&L and exit fields (often recording a TradeExitReason); if the trade is aborted before becoming active or explicitly stopped it moves to CANCELLED; DISABLED is used to mark trades that are intentionally prevented from progressing by risk rules or operator configuration. The pattern here mirrors the simple constant-enumeration approach used by OrderStatus and TradeExitReason, but TradeState is focused on the overall trade-level lifecycle (rather than individual order lifecycle or exit cause) and is the single attribute the rest of the system inspects to gate state transitions, persist final records and drive downstream reconciliation.

