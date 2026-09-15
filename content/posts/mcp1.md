+++
authors = ["Mike Harmon"]
title = "Agents, Tools, MCPs and All That: Part 1"
date = "2026-08-26"
tags = [
    "",
    "LangChain",
    "FastMCP",
    "NoSQL",
    "MongoDB",
    "AI",
    "Agents"
]
series = ["Agents"]
aliases = ["migrate-from-jekyl"]
+++

# Agents, Tools, MCPs and All That
---------

## 1. Introduction
-------------------

It's been a while since I posted. With my family commitments, writing is harder, but it's never been easier to work with LLMs and AI! The biggest developments in the past year or two have been agents and agentic AI applications. To be honest, I've always been a bit skeptical about agents and AI. In science we are taught to be skeptical, as Feynman said, ["The first principle is that you must not fool yourself—and you are the easiest person to fool."](http://brainyquote.com/quotes/richard_p_feynman_137642) However, over the last two years I've become an AI engineer and fully embraced AI.

In this post I discuss agents, tools, MCPs, and the surrounding ecosystem (the title riffs on the classic vector calculus book [*Div, Grad, Curl, and All That*](https://wwnorton.com/books/9780393925166)). I'll keep it brief and simple—long posts are harder to write, and attention spans are short.

I'll cover how to build a simple ReAct agent, use a [MCP](https://en.wikipedia.org/wiki/Model_Context_Protocol) server, and observe agent behavior using [LangChain](https://www.langchain.com/), [Groq](https://groq.com/), [FastMCP](https://gofastmcp.com/getting-started/welcome), and [LangSmith](https://www.langchain.com/langsmith-platform). The agent will have tools for retrieving weather (a classic first example) and locating the nearest police station and public restroom in NYC (data from [OpenData NYC](https://data.cityofnewyork.us/)). In the backend I'll use [MongoDB](https://www.mongodb.com/), [Redis](https://redis.io/), and several APIs to accomplish these tasks. I won't go into how the underlying functions work, but instead focus on how they can be used as tools.


```python
import sys
import os
from dotenv import load_dotenv, find_dotenv
load_dotenv(find_dotenv())

from langchain_groq import ChatGroq
from langchain_core.output_parsers import StrOutputParser
from langchain.agents import create_agent
```



## 2. Agents
-------------------
Large language models (LLMs) have been round for a few years at this point and most people are familiar with them through chatbots such as ChatGPT. With the advent of ChatGPT, people saw a AI model could have converations and write prose that was similar to a humans ability. LLMs were trained on an enormous amount of text data from the internet and can answer almost any question. I still remember in late 2022 asking ChatGPT to prove the uniqueness of solutions to the diffusion equation, it crushed the answer in 2 seconds. I asked more questiosn from semiconductor physics, quantum mechanics and numerical analysis, it got them all correct. I was impressed, but also realized it was reguritating things from the internet. Here's an example of a simple LLM with [LangChain](https://www.langchain.com/)


```python
model = ChatGroq(model="openai/gpt-oss-120b")
llm  = model | StrOutputParser() 

print(llm.invoke("Give me a 100 word explanation of what a Sobolev Space is."))
```

    A Sobolev space is a functional space that extends the concept of differentiability to functions whose derivatives may not be classically defined but exist in an integral sense. Formally, for an open set Ω⊂ℝⁿ and an integer k≥0, the Sobolev space W^{k,p}(Ω) consists of L^{p}(Ω) functions whose weak derivatives up to order k also belong to L^{p}(Ω). These spaces are Banach (Hilbert when p=2) and provide the natural setting for variational formulations of partial differential equations, embedding theorems, and regularity theory in practice. Sobolev spaces also enable trace theorems, allowing boundary values to be defined for functions lacking classical continuity.


Amazing right? 

Now ask it something simple:


```python
print(llm.invoke("What happened on July 4th 2026?"))
```

    I’m sorry, but I don’t have information about events that occurred on July 4, 2026. My training only includes data up through 2024, so I’m not able to provide details about that date. If you have a more specific question or need information about earlier years, I’d be happy to help.


While the model has compressed a vast amount of humanity's knowledge into 120 billion parameters, it doesn't know events that occurred after its training cutoff. An LLM deployed for employee queries also lacks knowledge of company‑specific information.

Engineers addressed this with Retrieval‑Augmented Generation (RAG), which requires an up‑to‑date knowledge base. More recently, tools that let the LLM take actions (e.g., web search) enable it to look up missing information and interact with its environment—editing files, summarizing emails, sending messages, and more. An agent is an LLM equipped with a set of tools. The simplest agent decides which steps to take based on a request and its internal state, using tools to act or generate text from the current context. This pattern is known as a [ReAct agent](https://arxiv.org/abs/2210.03629).

The ability of an LLM to both reason and act with tools is driving a technological revolution. We'll now cover the basics of tools.

## 3. Agents & Tools
-------------------
Let's go over first the tool everyone stars with, which is the ability to get the weather. A tool is a regular function; however it needs to be wrapped in a decorator to declare it a tool. We'll go over that in a second, but for now I'll import the function:


```python
PROJECT_ROOT = os.path.abspath('..')
sys.path.insert(0, PROJECT_ROOT)
from mymcp.server import get_weather
```

The defintion of this function is [here](https://github.com/mdh266/langchain-fastmcp/blob/main/mymcp/server.py). It is a simple function that takes in a city name and queries the [OpenWeather API](https://openweathermap.org/) to give us the current weather:


```python
get_weather("New York")
```




    {'coord': {'lon': -74.006, 'lat': 40.7143},
     'weather': [{'id': 800,
       'main': 'Clear',
       'description': 'clear sky',
       'icon': '01n'}],
     'base': 'stations',
     'main': {'temp': 25.12,
      'feels_like': 25.2,
      'temp_min': 23.21,
      'temp_max': 26.19,
      'pressure': 1011,
      'humidity': 58,
      'sea_level': 1011,
      'grnd_level': 1010},
     'visibility': 10000,
     'wind': {'speed': 5.66, 'deg': 250},
     'clouds': {'all': 0},
     'dt': 1787532946,
     'sys': {'type': 1,
      'id': 4610,
      'country': 'US',
      'sunrise': 1787480061,
      'sunset': 1787528603},
     'timezone': -14400,
     'id': 5128581,
     'name': 'New York',
     'cod': 200}



Now we can wrap the Python function with `tool("get_weather", get_weather)` to expose it to the LLM.


```python
from langchain.tools import tool
weather_tool = tool("get_weather", get_weather)
```

The tool is actually now a co-routine that can be called with the async-invoke method:


```python
await weather_tool.ainvoke("New York")
```




    {'coord': {'lon': -74.006, 'lat': 40.7143},
     'weather': [{'id': 800,
       'main': 'Clear',
       'description': 'clear sky',
       'icon': '01n'}],
     'base': 'stations',
     'main': {'temp': 25.12,
      'feels_like': 25.2,
      'temp_min': 23.21,
      'temp_max': 26.19,
      'pressure': 1011,
      'humidity': 58,
      'sea_level': 1011,
      'grnd_level': 1010},
     'visibility': 10000,
     'wind': {'speed': 5.66, 'deg': 250},
     'clouds': {'all': 0},
     'dt': 1787532946,
     'sys': {'type': 1,
      'id': 4610,
      'country': 'US',
      'sunrise': 1787480061,
      'sunset': 1787528603},
     'timezone': -14400,
     'id': 5128581,
     'name': 'New York',
     'cod': 200}



Now we can use the [create_agent](https://reference.langchain.com/python/langchain/agents/factory/create_agent) to create a simple ReAct agent:


```python
agents = create_agent(model=model, tools=[weather_tool])
```

Now I can pass the query in and use the agent using the same asynchronous `ainvoke` method,


```python
result = await agents.ainvoke({'messages': [{'role': 'user', 'content': "What is the weather in New York?"}]})
messages = result.get("messages")
```


The ReAct agent returns a list of the all messages in the conversation:


```python
messages
```




    [HumanMessage(content='What is the weather in New York?', additional_kwargs={}, response_metadata={}, id='9aa90351-2efe-4446-8350-f0b87e5d88c8'),
     AIMessage(content='', additional_kwargs={'reasoning_content': 'User asks: "What is the weather in New York?" Need to call get_weather function with city "New York". Use function.', 'tool_calls': [{'id': 'fc_1908a057-388e-4232-82d2-28c608f0632d', 'function': {'arguments': '{"city":"New York"}', 'name': 'get_weather'}, 'type': 'function'}]}, response_metadata={'token_usage': {'completion_tokens': 56, 'prompt_tokens': 261, 'total_tokens': 317, 'completion_time': 0.118889854, 'completion_tokens_details': {'reasoning_tokens': 28}, 'prompt_time': 0.022129046, 'prompt_tokens_details': None, 'queue_time': 0.078422765, 'total_time': 0.1410189}, 'model_name': 'openai/gpt-oss-120b', 'system_fingerprint': 'fp_0adba2bb92', 'service_tier': 'on_demand', 'finish_reason': 'tool_calls', 'logprobs': None, 'model_provider': 'groq'}, id='lc_run--01a0316d-2483-75e3-b1d4-2f407cae8cef-0', tool_calls=[{'name': 'get_weather', 'args': {'city': 'New York'}, 'id': 'fc_1908a057-388e-4232-82d2-28c608f0632d', 'type': 'tool_call'}], invalid_tool_calls=[], usage_metadata={'input_tokens': 261, 'output_tokens': 56, 'total_tokens': 317, 'output_token_details': {'reasoning': 28}}),
     ToolMessage(content='{"coord": {"lon": -74.006, "lat": 40.7143}, "weather": [{"id": 800, "main": "Clear", "description": "clear sky", "icon": "01n"}], "base": "stations", "main": {"temp": 24.47, "feels_like": 24.54, "temp_min": 22.65, "temp_max": 25.54, "pressure": 1012, "humidity": 60, "sea_level": 1012, "grnd_level": 1010}, "visibility": 10000, "wind": {"speed": 5.66, "deg": 250}, "clouds": {"all": 0}, "dt": 1787535581, "sys": {"type": 1, "id": 4610, "country": "US", "sunrise": 1787480061, "sunset": 1787528603}, "timezone": -14400, "id": 5128581, "name": "New York", "cod": 200}', name='get_weather', id='171d3e91-38a5-4f5b-bd8a-6bb88f3ac16c', tool_call_id='fc_1908a057-388e-4232-82d2-28c608f0632d'),
     AIMessage(content='**Current weather in New\u202fYork (as of the latest report):**\n\n| Parameter | Value |\n|-----------|-------|\n| **Condition** | Clear sky |\n| **Temperature** | **24\u202f°C** (feels like 24\u202f°C) |\n| **Humidity** | 60\u202f% |\n| **Wind** | 5.7\u202fm/s from the west‑south‑west (≈\u202f250°) |\n| **Pressure** | 1012\u202fhPa |\n| **Visibility** | 10\u202fkm |\n| **Cloud cover** | 0\u202f% |\n\n*The data comes from OpenWeatherMap and reflects the situation at the time of the API call.*', additional_kwargs={'reasoning_content': 'We need to present weather info. Convert temperature from Kelvin? Actually API likely returns Kelvin? The temp 24.47 suggests Celsius maybe. Could be Kelvin? 24.47°C is plausible. Provide description, temperature, feels like, humidity, wind speed. Also note time conversion. Provide user-friendly answer.'}, response_metadata={'token_usage': {'completion_tokens': 215, 'prompt_tokens': 525, 'total_tokens': 740, 'completion_time': 0.443993506, 'completion_tokens_details': {'reasoning_tokens': 64}, 'prompt_time': 0.022505999, 'prompt_tokens_details': None, 'queue_time': 0.173320188, 'total_time': 0.466499505}, 'model_name': 'openai/gpt-oss-120b', 'system_fingerprint': 'fp_96d96a151c', 'service_tier': 'on_demand', 'finish_reason': 'stop', 'logprobs': None, 'model_provider': 'groq'}, id='lc_run--01a0316d-2641-75d3-90f3-20b481ced882-0', tool_calls=[], invalid_tool_calls=[], usage_metadata={'input_tokens': 525, 'output_tokens': 215, 'total_tokens': 740, 'output_token_details': {'reasoning': 64}})]



The conversation flow is HumanMessage → AIMessage (reasoning) → ToolMessage → AIMessage (final answer). For longer traces, LangSmith visualises these steps which makes them easier to follow. 

## 4. Model Context Protocols (MCPs)
-------------------

Tools are often tied to local code, which makes sharing hard. Additionally, each LLM and agent framework (OpenAI, Google, Anthropic, etc.) has their own way of defining and using tools. This can cause headaches for developers that need to make tools work across these various frameworks. The [**Model Context Protocol (MCP)**](https://en.wikipedia.org/wiki/Model_Context_Protocol) is an open standard for connecting LLM applications to external tools and data sources.

MCP provides a common interface by creating,

- **MCP servers**: Exposes tools, resources, or prompts.
- **MCP clients**: Connects an application or agent to the server.
- **Schemas**: Describes the tool's arguments and return values.

Model context protocol improves upon local tools by offering standard protocols for using tools on a remote server. Agents do not need direct access to data or source code the tool uses. In some ways, MCP is for agents as the way APIs are for regular code. The standard MCP framework is [FastMCP](https://gofastmcp.com/getting-started/welcome). I created a FastMCP server here with several tools I'll discuss. 

First, we'll create an MCP client and connect to our locally running server:


```python
from fastmcp import Client
mcp_client = Client("http://localhost:8080/mcp")
```

Then we can get a list of the tools,


```python
async with mcp_client:
    tools = await mcp_client.list_tools()

tools
```




    [Tool(name='get_weather', title=None, description='Fetch current weather for *city* from OpenWeatherMap.\n\nThe function reads the API key from the ``OPEN_WEATHER_MAP_API_KEY``\nenvironment variable unless an explicit ``api_key`` argument is supplied.', inputSchema={'additionalProperties': False, 'properties': {'city': {'type': 'string', 'description': 'City name to query, e.g. ``"London"``.'}, 'api_key': {'anyOf': [{'type': 'string'}, {'type': 'null'}], 'default': None, 'description': 'Optional explicit API key; if omitted the environment variable\nis used.'}}, 'required': ['city'], 'type': 'object'}, outputSchema={'additionalProperties': True, 'type': 'object'}, icons=None, annotations=None, meta={'fastmcp': {'tags': []}}, execution=None),
     Tool(name='convert_address_to_point', title=None, description='Convert a street address to a Shapely Point object.', inputSchema={'additionalProperties': False, 'properties': {'address': {'type': 'string', 'description': 'A free‑form US address (e.g. ``"123 Main St, New York, NY 10001"``).'}}, 'required': ['address'], 'type': 'object'}, outputSchema={'additionalProperties': {'type': 'number'}, 'type': 'object'}, icons=None, annotations=None, meta={'fastmcp': {'tags': []}}, execution=None),
     Tool(name='get_police_precinct', title=None, description='Resolve a street address to the NYPD precinct number.', inputSchema={'additionalProperties': False, 'properties': {'lat': {'type': 'number', 'description': 'Latitude of the location.'}, 'lng': {'type': 'number', 'description': 'Longitude of the location.'}}, 'required': ['lat', 'lng'], 'type': 'object'}, outputSchema={'properties': {'result': {'anyOf': [{'type': 'integer'}, {'type': 'null'}]}}, 'required': ['result'], 'type': 'object', 'x-fastmcp-wrap-result': True}, icons=None, annotations=None, meta={'fastmcp': {'tags': []}}, execution=None),
     Tool(name='find_closest_restroom', title=None, description='Find the closest public restroom to a given latitude and longitude and return all information on it', inputSchema={'additionalProperties': False, 'properties': {'lat': {'type': 'number', 'description': 'Latitude of the location to search from.'}, 'lng': {'type': 'number', 'description': 'Longitude of the location to search from.'}}, 'required': ['lat', 'lng'], 'type': 'object'}, outputSchema={'type': 'object', 'additionalProperties': True}, icons=None, annotations=None, meta={'fastmcp': {'tags': []}}, execution=None),
     Tool(name='get_precinct_info', title=None, description='Return the precinct information for a given precinct number.', inputSchema={'additionalProperties': False, 'properties': {'precinct_number': {'type': 'integer', 'description': 'The precinct number (e.g. 1, 5, 14, etc.).'}}, 'required': ['precinct_number'], 'type': 'object'}, outputSchema={'additionalProperties': {'type': 'string'}, 'type': 'object'}, icons=None, annotations=None, meta={'fastmcp': {'tags': []}}, execution=None)]



These tools are of type [FastMCP tools](https://gofastmcp.com/servers/tools),


```python
type(tools[0])
```




    mcp.types.Tool



We can then call a specific tool using the client's [call_tool](https://gofastmcp.com/clients/tools#execution-options) method


```python
async with mcp_client:
    result = await mcp_client.call_tool("get_weather", {"city": "New York"})
```


```python
print(result)
```

    CallToolResult(content=[TextContent(type='text', text='{"coord":{"lon":-74.006,"lat":40.7143},"weather":[{"id":803,"main":"Clouds","description":"broken clouds","icon":"04n"}],"base":"stations","main":{"temp":24.3,"feels_like":24.74,"temp_min":22.65,"temp_max":25.14,"pressure":1017,"humidity":75,"sea_level":1017,"grnd_level":1016},"visibility":10000,"wind":{"speed":4.47,"deg":163,"gust":5.36},"clouds":{"all":52},"dt":1787790329,"sys":{"type":1,"id":4610,"country":"US","sunrise":1787739438,"sunset":1787787529},"timezone":-14400,"id":5128581,"name":"New York","cod":200}', annotations=None, meta=None)], structured_content={'coord': {'lon': -74.006, 'lat': 40.7143}, 'weather': [{'id': 803, 'main': 'Clouds', 'description': 'broken clouds', 'icon': '04n'}], 'base': 'stations', 'main': {'temp': 24.3, 'feels_like': 24.74, 'temp_min': 22.65, 'temp_max': 25.14, 'pressure': 1017, 'humidity': 75, 'sea_level': 1017, 'grnd_level': 1016}, 'visibility': 10000, 'wind': {'speed': 4.47, 'deg': 163, 'gust': 5.36}, 'clouds': {'all': 52}, 'dt': 1787790329, 'sys': {'type': 1, 'id': 4610, 'country': 'US', 'sunrise': 1787739438, 'sunset': 1787787529}, 'timezone': -14400, 'id': 5128581, 'name': 'New York', 'cod': 200}, meta=None, data={'coord': {'lon': -74.006, 'lat': 40.7143}, 'weather': [{'id': 803, 'main': 'Clouds', 'description': 'broken clouds', 'icon': '04n'}], 'base': 'stations', 'main': {'temp': 24.3, 'feels_like': 24.74, 'temp_min': 22.65, 'temp_max': 25.14, 'pressure': 1017, 'humidity': 75, 'sea_level': 1017, 'grnd_level': 1016}, 'visibility': 10000, 'wind': {'speed': 4.47, 'deg': 163, 'gust': 5.36}, 'clouds': {'all': 52}, 'dt': 1787790329, 'sys': {'type': 1, 'id': 4610, 'country': 'US', 'sunrise': 1787739438, 'sunset': 1787787529}, 'timezone': -14400, 'id': 5128581, 'name': 'New York', 'cod': 200}, is_error=False)


Notice that it returns a [CallToolResult](https://gofastmcp.com/clients/tools#structured-results). This is all pretty much the standard example, so I'll show a unique too, which is finding the closest public restroom to the point we received back:


```python
lon = result.data.get("coord").get("lon")
lat = result.data.get("coord").get("lat")

async with mcp_client:
    restroom = await mcp_client.call_tool("find_closest_restroom", {"lat": lat, "lng": lon})
```


```python
restroom.data
```




    {'facility_name': 'New Amsterdam Library, NYPL',
     'location_type': 'Library',
     'operator': 'NYPL',
     'status': 'Operational',
     'open': 'Year Round',
     'hours_of_operation': 'Sunday: Closed \nMonday: 10:00 am - 7:00 pm \nTuesday: 10:00 am - 7:00 pm  \nWednesday: 10:00 am - 7:00 pm   \nThursday: 10:00 am - 7:00 pm   \nFriday: 10:00 am - 5:00 pm \nSaturday: 10:00 am - 5:00 pm',
     'accessibility': 'Fully Accessible',
     'restroom_type': 'Single-Stall All Gender Restroom(s)',
     'changing_stations': 'Yes',
     'latitude': None,
     'longitude': None,
     'website': 'https://www.nypl.org/locations/new-amsterdam'}



Great! Next let's give a LangChain agent access to these tools!

## 5. Agents & MCPs
-------------------

Now in order to make an LangChain agent have access to the tools supplied by an MCP server we need to use a specialized MCP client for LangChain called [LangChain MCP Adapter](https://reference.langchain.com/python/langchain-mcp-adapters). This can be imported and set to the address of the MCP server and set transport type:


```python
from langchain_mcp_adapters.client import MultiServerMCPClient  

lc_client = MultiServerMCPClient({"config": {"url": "http://localhost:8080/mcp", "transport": "http"}})
```

Now we can get the tools:


```python
tool_list = await lc_client.get_tools()
tool_list
```




    [StructuredTool(name='get_weather', description='Fetch current weather for *city* from OpenWeatherMap.\n\nThe function reads the API key from the ``OPEN_WEATHER_MAP_API_KEY``\nenvironment variable unless an explicit ``api_key`` argument is supplied.', args_schema={'additionalProperties': False, 'properties': {'city': {'type': 'string', 'description': 'City name to query, e.g. ``"London"``.'}, 'api_key': {'anyOf': [{'type': 'string'}, {'type': 'null'}], 'default': None, 'description': 'Optional explicit API key; if omitted the environment variable\nis used.'}}, 'required': ['city'], 'type': 'object'}, metadata={'_meta': {'fastmcp': {'tags': []}}}, response_format='content_and_artifact', coroutine=<function convert_mcp_tool_to_langchain_tool.<locals>.call_tool at 0x12657fb00>),
     StructuredTool(name='convert_address_to_point', description='Convert a street address to a Shapely Point object.', args_schema={'additionalProperties': False, 'properties': {'address': {'type': 'string', 'description': 'A free‑form US address (e.g. ``"123 Main St, New York, NY 10001"``).'}}, 'required': ['address'], 'type': 'object'}, metadata={'_meta': {'fastmcp': {'tags': []}}}, response_format='content_and_artifact', coroutine=<function convert_mcp_tool_to_langchain_tool.<locals>.call_tool at 0x126320b80>),
     StructuredTool(name='get_police_precinct', description='Resolve a street address to the NYPD precinct number.', args_schema={'additionalProperties': False, 'properties': {'lat': {'type': 'number', 'description': 'Latitude of the location.'}, 'lng': {'type': 'number', 'description': 'Longitude of the location.'}}, 'required': ['lat', 'lng'], 'type': 'object'}, metadata={'_meta': {'fastmcp': {'tags': []}}}, response_format='content_and_artifact', coroutine=<function convert_mcp_tool_to_langchain_tool.<locals>.call_tool at 0x12072a840>),
     StructuredTool(name='find_closest_restroom', description='Find the closest public restroom to a given latitude and longitude and return all information on it', args_schema={'additionalProperties': False, 'properties': {'lat': {'type': 'number', 'description': 'Latitude of the location to search from.'}, 'lng': {'type': 'number', 'description': 'Longitude of the location to search from.'}}, 'required': ['lat', 'lng'], 'type': 'object'}, metadata={'_meta': {'fastmcp': {'tags': []}}}, response_format='content_and_artifact', coroutine=<function convert_mcp_tool_to_langchain_tool.<locals>.call_tool at 0x12072bf60>),
     StructuredTool(name='get_precinct_info', description='Return the precinct information for a given precinct number.', args_schema={'additionalProperties': False, 'properties': {'precinct_number': {'type': 'integer', 'description': 'The precinct number (e.g. 1, 5, 14, etc.).'}}, 'required': ['precinct_number'], 'type': 'object'}, metadata={'_meta': {'fastmcp': {'tags': []}}}, response_format='content_and_artifact', coroutine=<function convert_mcp_tool_to_langchain_tool.<locals>.call_tool at 0x12072bb00>)]



Notice these are [LangChain tools](https://docs.langchain.com/oss/python/langchain/tools)! The adapter converted the FastMCP tool to a LangChain one:


```python
type(tool_list[0])
```




    langchain_core.tools.structured.StructuredTool



We can then use the tool as before,


```python
temperature = await tool_list[0].ainvoke({"city":"Boston"})
```

The immediate return value isnt quite the same, its a list with json in it. We can convert back though,


```python
import json
json.loads(temperature[0].get("text"))
```




    {'coord': {'lon': -71.0598, 'lat': 42.3584},
     'weather': [{'id': 804,
       'main': 'Clouds',
       'description': 'overcast clouds',
       'icon': '04n'}],
     'base': 'stations',
     'main': {'temp': 22.4,
      'feels_like': 22.65,
      'temp_min': 20.64,
      'temp_max': 23.8,
      'pressure': 1017,
      'humidity': 75,
      'sea_level': 1017,
      'grnd_level': 1014},
     'visibility': 10000,
     'wind': {'speed': 2.57, 'deg': 140},
     'clouds': {'all': 86},
     'dt': 1787791539,
     'sys': {'type': 2,
      'id': 2007536,
      'country': 'US',
      'sunrise': 1787738592,
      'sunset': 1787786960},
     'timezone': -14400,
     'id': 4930956,
     'name': 'Boston',
     'cod': 200}



Now let's give the agent access to these tools from the MCP server. This just like giving the agent any other tools, 


```python
agent = create_agent(model=model, tools=tool_list)
```

And now the agent can answer my question on the weather:


```python
result = await agent.ainvoke({
    'messages': [{'role': 'user', 
                 'content': 'Whats the temparture in Boston?'
}]})
print(result.get("messages")[-1].content)
```

    The current temperature in Boston is **about 22 °C** (≈ 72 °F).


Great!

Let's try something less straight forward. The [tool to find the police precinct](https://github.com/mdh266/langchain-fastmcp/blob/main/mymcp/server.py#L89) requires a latitude and longitude. If a users passes in an address the agent would need to conver the address to a latitude and longitude ([convert_address_to_point](https://github.com/mdh266/langchain-fastmcp/blob/main/mymcp/server.py#L57) tool) first and then pass that to the [get_police_precint tool](https://github.com/mdh266/langchain-fastmcp/blob/main/mymcp/server.py#L87). We can see this plainly with the entire message chain:


```python
result = await agent.ainvoke({
    'messages': [{'role': 'user', 
                 'content': 'What police precinct is 306 West 54th Street, manhattan, manhattan in?'
}]})
```


```python
result.get("messages")
```




    [HumanMessage(content='What police precinct is 306 West 54th Street, manhattan, manhattan in?', additional_kwargs={}, response_metadata={}, id='3fa87a68-b890-4b89-8a54-dfafc867ea54'),
     AIMessage(content='', additional_kwargs={'reasoning_content': 'The user asks: "What police precinct is 306 West 54th Street, manhattan, manhattan in?" Need to determine precinct number. We have functions: convert_address_to_point, get_police_precinct. So first convert address to point, then get precinct. Use convert_address_to_point with address string. Then feed lat/lng to get_police_precinct. Then respond with precinct number and maybe info. Could also fetch precinct info using get_precinct_info for more detail. Let\'s do steps.\n\n', 'tool_calls': [{'id': 'fc_3d5d66e2-d976-410b-a5d2-ebc205032363', 'function': {'arguments': '{"address":"306 West 54th Street, Manhattan, NY"}', 'name': 'convert_address_to_point'}, 'type': 'function'}]}, response_metadata={'token_usage': {'completion_tokens': 143, 'prompt_tokens': 432, 'total_tokens': 575, 'completion_time': 0.300365324, 'completion_tokens_details': {'reasoning_tokens': 105}, 'prompt_time': 0.083295298, 'prompt_tokens_details': None, 'queue_time': 0.092930886, 'total_time': 0.383660622}, 'model_name': 'openai/gpt-oss-120b', 'system_fingerprint': 'fp_90620edd96', 'service_tier': 'on_demand', 'finish_reason': 'tool_calls', 'logprobs': None, 'model_provider': 'groq'}, id='lc_run--01a040b9-e3af-7960-957e-7e05f42d8720-0', tool_calls=[{'name': 'convert_address_to_point', 'args': {'address': '306 West 54th Street, Manhattan, NY'}, 'id': 'fc_3d5d66e2-d976-410b-a5d2-ebc205032363', 'type': 'tool_call'}], invalid_tool_calls=[], usage_metadata={'input_tokens': 432, 'output_tokens': 143, 'total_tokens': 575, 'output_token_details': {'reasoning': 105}}),
     ToolMessage(content=[{'type': 'text', 'text': '{"lat":40.7649719,"lng":-73.98510619999999}', 'id': 'lc_6a9efcb3-1cea-4c1f-941e-27e05b66386d'}], name='convert_address_to_point', id='0f39cebb-b07f-4a5b-af16-0a39c85ce5c9', tool_call_id='fc_3d5d66e2-d976-410b-a5d2-ebc205032363', artifact={'structured_content': {'lat': 40.7649719, 'lng': -73.98510619999999}}),
     AIMessage(content='', additional_kwargs={'reasoning_content': 'We have lat/lng. Now need precinct. Use get_police_precinct.', 'tool_calls': [{'id': 'fc_c2bdf7ab-5136-4fe0-93ed-59aadcb2515c', 'function': {'arguments': '{"lat":40.7649719,"lng":-73.98510619999999}', 'name': 'get_police_precinct'}, 'type': 'function'}]}, response_metadata={'token_usage': {'completion_tokens': 59, 'prompt_tokens': 491, 'total_tokens': 550, 'completion_time': 0.129608512, 'completion_tokens_details': {'reasoning_tokens': 18}, 'prompt_time': 0.021661499, 'prompt_tokens_details': None, 'queue_time': 0.083568145, 'total_time': 0.151270011}, 'model_name': 'openai/gpt-oss-120b', 'system_fingerprint': 'fp_017482bd7f', 'service_tier': 'on_demand', 'finish_reason': 'tool_calls', 'logprobs': None, 'model_provider': 'groq'}, id='lc_run--01a040b9-e828-7661-b582-f01df67f93ee-0', tool_calls=[{'name': 'get_police_precinct', 'args': {'lat': 40.7649719, 'lng': -73.98510619999999}, 'id': 'fc_c2bdf7ab-5136-4fe0-93ed-59aadcb2515c', 'type': 'tool_call'}], invalid_tool_calls=[], usage_metadata={'input_tokens': 491, 'output_tokens': 59, 'total_tokens': 550, 'output_token_details': {'reasoning': 18}}),
     ToolMessage(content=[{'type': 'text', 'text': '18', 'id': 'lc_c458cb91-f7a1-4f7e-9492-bd4dc777cf4b'}], name='get_police_precinct', id='52c0df29-db9a-4a91-ba25-b93b849cf440', tool_call_id='fc_c2bdf7ab-5136-4fe0-93ed-59aadcb2515c', artifact={'structured_content': {'result': 18}}),
     AIMessage(content='The address **306\u202fWest\u202f54th\u202fStreet, Manhattan, NY** falls within **NYPD Precinct\u202f18**.', additional_kwargs={'reasoning_content': 'We have precinct number 18. Need to answer.'}, response_metadata={'token_usage': {'completion_tokens': 49, 'prompt_tokens': 539, 'total_tokens': 588, 'completion_time': 0.101065899, 'completion_tokens_details': {'reasoning_tokens': 12}, 'prompt_time': 0.021749634, 'prompt_tokens_details': None, 'queue_time': 0.243815915, 'total_time': 0.122815533}, 'model_name': 'openai/gpt-oss-120b', 'system_fingerprint': 'fp_6dedd2be22', 'service_tier': 'on_demand', 'finish_reason': 'stop', 'logprobs': None, 'model_provider': 'groq'}, id='lc_run--01a040b9-ed3f-7820-882d-5152d5f2cf42-0', tool_calls=[], invalid_tool_calls=[], usage_metadata={'input_tokens': 539, 'output_tokens': 49, 'total_tokens': 588, 'output_token_details': {'reasoning': 12}})]



We can see that agent reasoning:


```python
print(result.get("messages")[1].additional_kwargs.get("reasoning_content"))
```

    The user asks: "What police precinct is 306 West 54th Street, manhattan, manhattan in?" Need to determine precinct number. We have functions: convert_address_to_point, get_police_precinct. So first convert address to point, then get precinct. Use convert_address_to_point with address string. Then feed lat/lng to get_police_precinct. Then respond with precinct number and maybe info. Could also fetch precinct info using get_precinct_info for more detail. Let's do steps.
    
    


One thing to note is that the MCP tools take up space in the model context window which causes challenges. There are strategies to mitigate this, but I wont go over this here. The final message from the agent is,


```python
print(result.get("messages")[-1].content)
```

    The address **306 West 54th Street, Manhattan, NY** falls within **NYPD Precinct 18**.


Looking at the agent's actions in this way, a list of messages, is not ideal. This is where [LangSmith](https://www.langchain.com/langsmith-platform) comes into play.

<!--  -->

## 6. Agent Observability With Langsmith
-------------------

Parsing throught list of message is less than ideal. It also doesn't help us understand where and error occurred or what part of the process took the most amount of time. LangSmith offers as a way to do all of this through there interface. LangSmith integrates with most agent harnesses and integreates seamlessly with LangChain and LangGraph.

As an example I can force the LLM to take 3 tool calls to get me the phone number of the closest police precinct (this requires the precinct number be sent to [get_precinct_info](https://github.com/mdh266/langchain-fastmcp/blob/main/mymcp/server.py#L207C5-L207C22) tool). The query is:


```python
result = await agent.ainvoke({
    'messages': [{'role': 'user', 
                 'content': 'What is the phone number for closests police station to 306 West 54th Street, Manhattan?'
}]})
```

The agent traces on LangSmith show us the path,

<img src="https://github.com/mdh266/langchain-fastmcp/blob/main/images/trace1.png?raw=1">

The agent clearly goes from `query` -> `convert_address_to_point` -> `get_police_precinct` -> `get_precinct_info`.

Zooming into the last tool call, we can see that the agent figured out how to pass the police precinct number into the tool:

<img src="https://github.com/mdh266/langchain-fastmcp/blob/main/images/trace2.png?raw=1">

The agent receives the json result back from the tool and then generates a reponse that we see as,


```python
print(result.get("messages")[-1].content)
```

    The nearest NYPD precinct to 306 West 54th Street, Manhattan is the **Midtown North Precinct (Precinct 18)**.  
    
    **Phone:** **212‑767‑8400**.


## 7. Next Steps
-------

In this post I went over how to create agents with Langchain and give them access to tools through FastMCP. Additionally we covered loggin the agent traces using LangSmith. This was all pretty simple proeject and the MCP Server is local. In a follow up post I'll show how to deploy this MCP server to Google Cloud and add it to Claude Code or another coding agent.  

Hope you enjoyed this!
