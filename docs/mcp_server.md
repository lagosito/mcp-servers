Your First MCP Server
TypeScript
Create a simple MCP server in TypeScript in 15 minutes

Let’s build your first MCP server in TypeScript! We’ll create a weather server that provides current weather data as a resource and lets Claude fetch forecasts using tools.

This guide uses the OpenWeatherMap API. You’ll need a free API key from OpenWeatherMap to follow along.

​
Prerequisites
1
Install Node.js

You’ll need Node.js 18 or higher:


node --version  # Should be v18 or higher
npm --version
2
Create a new project

You can use our create-typescript-server tool to bootstrap a new project:


npx @modelcontextprotocol/create-server weather-server
cd weather-server
3
Install dependencies


npm install --save axios dotenv
4
Set up environment

Create .env:


OPENWEATHER_API_KEY=your-api-key-here
Make sure to add your environment file to .gitignore


.env
​
Create your server
1
Define types

Create a file src/types.ts, and add the following:


export interface OpenWeatherResponse {
  main: {
    temp: number;
    humidity: number;
  };
  weather: Array<{
    description: string;
  }>;
  wind: {
    speed: number;
  };
  dt_txt?: string;
}

export interface WeatherData {
  temperature: number;
  conditions: string;
  humidity: number;
  wind_speed: number;
  timestamp: string;
}

export interface ForecastDay {
  date: string;
  temperature: number;
  conditions: string;
}

export interface GetForecastArgs {
  city: string;
  days?: number;
}

// Type guard for forecast arguments
export function isValidForecastArgs(args: any): args is GetForecastArgs {
  return (
    typeof args === "object" && 
    args !== null && 
    "city" in args &&
    typeof args.city === "string" &&
    (args.days === undefined || typeof args.days === "number")
  );
}
2
Add the base code

Replace src/index.ts with the following:


#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
  ListToolsRequestSchema,
  CallToolRequestSchema,
  ErrorCode,
  McpError
} from "@modelcontextprotocol/sdk/types.js";
import axios from "axios";
import dotenv from "dotenv";
import { 
  WeatherData, 
  ForecastDay, 
  OpenWeatherResponse,
  isValidForecastArgs 
} from "./types.js";

dotenv.config();

const API_KEY = process.env.OPENWEATHER_API_KEY;
if (!API_KEY) {
  throw new Error("OPENWEATHER_API_KEY environment variable is required");
}

const API_CONFIG = {
  BASE_URL: 'http://api.openweathermap.org/data/2.5',
  DEFAULT_CITY: 'San Francisco',
  ENDPOINTS: {
    CURRENT: 'weather',
    FORECAST: 'forecast'
  }
} as const;

class WeatherServer {
  private server: Server;
  private axiosInstance;

  constructor() {
    this.server = new Server({
      name: "example-weather-server",
      version: "0.1.0"
    }, {
      capabilities: {
        resources: {},
        tools: {}
      }
    });

    // Configure axios with defaults
    this.axiosInstance = axios.create({
      baseURL: API_CONFIG.BASE_URL,
      params: {
        appid: API_KEY,
        units: "metric"
      }
    });

    this.setupHandlers();
    this.setupErrorHandling();
  }

  private setupErrorHandling(): void {
    this.server.onerror = (error) => {
      console.error("[MCP Error]", error);
    };

    process.on('SIGINT', async () => {
      await this.server.close();
      process.exit(0);
    });
  }

  private setupHandlers(): void {
    this.setupResourceHandlers();
    this.setupToolHandlers();
  }

  private setupResourceHandlers(): void {
    // Implementation continues in next section
  }

  private setupToolHandlers(): void {
    // Implementation continues in next section
  }

  async run(): Promise<void> {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    
    // Although this is just an informative message, we must log to stderr,
    // to avoid interfering with MCP communication that happens on stdout
    console.error("Weather MCP server running on stdio");
  }
}

const server = new WeatherServer();
server.run().catch(console.error);
3
Add resource handlers

Add this to the setupResourceHandlers method:


private setupResourceHandlers(): void {
  this.server.setRequestHandler(
    ListResourcesRequestSchema,
    async () => ({
      resources: [{
        uri: `weather://${API_CONFIG.DEFAULT_CITY}/current`,
        name: `Current weather in ${API_CONFIG.DEFAULT_CITY}`,
        mimeType: "application/json",
        description: "Real-time weather data including temperature, conditions, humidity, and wind speed"
      }]
    })
  );

  this.server.setRequestHandler(
    ReadResourceRequestSchema,
    async (request) => {
      const city = API_CONFIG.DEFAULT_CITY;
      if (request.params.uri !== `weather://${city}/current`) {
        throw new McpError(
          ErrorCode.InvalidRequest,
          `Unknown resource: ${request.params.uri}`
        );
      }

      try {
        const response = await this.axiosInstance.get<OpenWeatherResponse>(
          API_CONFIG.ENDPOINTS.CURRENT,
          {
            params: { q: city }
          }
        );

        const weatherData: WeatherData = {
          temperature: response.data.main.temp,
          conditions: response.data.weather[0].description,
          humidity: response.data.main.humidity,
          wind_speed: response.data.wind.speed,
          timestamp: new Date().toISOString()
        };

        return {
          contents: [{
            uri: request.params.uri,
            mimeType: "application/json",
            text: JSON.stringify(weatherData, null, 2)
          }]
        };
      } catch (error) {
        if (axios.isAxiosError(error)) {
          throw new McpError(
            ErrorCode.InternalError,
            `Weather API error: ${error.response?.data.message ?? error.message}`
          );
        }
        throw error;
      }
    }
  );
}
4
Add tool handlers

Add these handlers to the setupToolHandlers method:


private setupToolHandlers(): void {
  this.server.setRequestHandler(
    ListToolsRequestSchema,
    async () => ({
      tools: [{
        name: "get_forecast",
        description: "Get weather forecast for a city",
        inputSchema: {
          type: "object",
          properties: {
            city: {
              type: "string",
              description: "City name"
            },
            days: {
              type: "number",
              description: "Number of days (1-5)",
              minimum: 1,
              maximum: 5
            }
          },
          required: ["city"]
        }
      }]
    })
  );

  this.server.setRequestHandler(
    CallToolRequestSchema,
    async (request) => {
      if (request.params.name !== "get_forecast") {
        throw new McpError(
          ErrorCode.MethodNotFound,
          `Unknown tool: ${request.params.name}`
        );
      }

      if (!isValidForecastArgs(request.params.arguments)) {
        throw new McpError(
          ErrorCode.InvalidParams,
          "Invalid forecast arguments"
        );
      }

      const city = request.params.arguments.city;
      const days = Math.min(request.params.arguments.days || 3, 5);

      try {
        const response = await this.axiosInstance.get<{
          list: OpenWeatherResponse[]
        }>(API_CONFIG.ENDPOINTS.FORECAST, {
          params: {
            q: city,
            cnt: days * 8 // API returns 3-hour intervals
          }
        });

        const forecasts: ForecastDay[] = [];
        for (let i = 0; i < response.data.list.length; i += 8) {
          const dayData = response.data.list[i];
          forecasts.push({
            date: dayData.dt_txt?.split(' ')[0] ?? new Date().toISOString().split('T')[0],
            temperature: dayData.main.temp,
            conditions: dayData.weather[0].description
          });
        }

        return {
          content: {
            mimeType: "application/json",
            text: JSON.stringify(forecasts, null, 2)
          }
        };
      } catch (error) {
        if (axios.isAxiosError(error)) {
          return {
            content: {
              mimeType: "text/plain",
              text: `Weather API error: ${error.response?.data.message ?? error.message}`
            },
            isError: true,
          }
        }
        throw error;
      }
    }
  );
}
5
Build and test


npm run build
npm link
​
Connect to Claude Desktop
1
Update Claude config

If you didn’t already connect to Claude Desktop during project setup, add to claude_desktop_config.json:


{
  "mcpServers": {
    "weather": {
      "command": "weather-server",
      "env": {
        "OPENWEATHER_API_KEY": "your-api-key",
      }
    }
  }
}
2
Restart Claude

Quit Claude completely
Start Claude again
Look for your weather server in the 🔌 menu
​
Try it out!

Check Current Weather


Get a Forecast


Compare Weather

​
Understanding the code
Type Safety
Resources
Tools

interface WeatherData {
  temperature: number;
  conditions: string;
  humidity: number;
  wind_speed: number;
  timestamp: string;
}
TypeScript adds type safety to our MCP server, making it more reliable and easier to maintain.

​
Best practices
Error Handling
When a tool encounters an error, return the error message with isError: true, so the model can self-correct:


try {
  const response = await axiosInstance.get(...);
} catch (error) {
  if (axios.isAxiosError(error)) {
    return {
      content: {
        mimeType: "text/plain",
        text: `Weather API error: ${error.response?.data.message ?? error.message}`
      },
      isError: true,
    }
  }
  throw error;
}
For other handlers, throw an error, so the application can notify the user:


try {
  const response = await this.axiosInstance.get(...);
} catch (error) {
  if (axios.isAxiosError(error)) {
    throw new McpError(
      ErrorCode.InternalError,
      `Weather API error: ${error.response?.data.message}`
    );
  }
  throw error;
}
Type Validation

function isValidForecastArgs(args: any): args is GetForecastArgs {
  return (
    typeof args === "object" && 
    args !== null && 
    "city" in args &&
    typeof args.city === "string"
  );
}
You can also use libraries like Zod to perform this validation automatically.
​
Available transports
While this guide uses stdio to run the MCP server as a local process, MCP supports other transports as well.

​
Troubleshooting
The following troubleshooting tips are for macOS. Guides for other platforms are coming soon.

​
Build errors

# Check TypeScript version
npx tsc --version

# Clean and rebuild
rm -rf build/
npm run build
​
Runtime errors
Look for detailed error messages in the Claude Desktop logs:


# Monitor logs
tail -n 20 -f ~/Library/Application\ Support/Claude/mcp*.log
​
Type errors

# Check types without building
npx tsc --noEmit
​
