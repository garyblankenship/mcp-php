# Setting Up a Model Context Protocol (MCP) Server in Laravel

## Overview

This tutorial will guide you through creating a Laravel-based MCP server that can communicate with Large Language Models (LLMs) like Claude. The server will support both STDIO and SSE transport layers.

## Prerequisites

- Laravel 10.x
- PHP 8.1+
- Composer

## Step 1: Project Setup

```bash
composer create-project laravel/laravel mcp-server
cd mcp-server
```

## Step 2: Create Core Interfaces

First, create the basic interfaces that define our MCP components:

```bash
mkdir -p app/Mcp/Interfaces
```

Create these interface files:

1. `app/Mcp/Interfaces/ServerInterface.php`:
```php
<?php

namespace App\Mcp\Interfaces;

interface ServerInterface
{
    public function setCapabilities(array $capabilities): void;
    public function setRequestHandler(string $type, callable $handler): void;
    public function connect(TransportInterface $transport): void;
    public function close(): void;
    public function onError(\Throwable $error): void;
}
```

2. `app/Mcp/Interfaces/TransportInterface.php`:
```php
<?php

namespace App\Mcp\Interfaces;

interface TransportInterface
{
    public function readMessage(): ?array;
    public function writeMessage(array $message): void;
    public function close(): void;
}
```

3. `app/Mcp/Interfaces/ResourceInterface.php`:
```php
<?php

namespace App\Mcp\Interfaces;

interface ResourceInterface
{
    public function getUri(): string;
    public function getName(): string;
    public function getMimeType(): string;
    public function getDescription(): string;
    public function getContents(): array;
}
```

4. `app/Mcp/Interfaces/ToolInterface.php`:
```php
<?php

namespace App\Mcp\Interfaces;

interface ToolInterface
{
    public function getName(): string;
    public function getDescription(): string;
    public function getInputSchema(): array;
    public function execute(array $arguments): array;
}
```

## Step 3: Implement Transport Layer

Create transport implementations:

1. `app/Mcp/Transport/StdioTransport.php`:
```php
<?php

namespace App\Mcp\Transport;

use App\Mcp\Interfaces\TransportInterface;

class StdioTransport implements TransportInterface
{
    private $stdin;
    private $stdout;

    public function __construct()
    {
        $this->stdin = fopen('php://stdin', 'rb');
        $this->stdout = fopen('php://stdout', 'wb');
        while (ob_get_level()) ob_end_clean();
    }

    public function readMessage(): ?array
    {
        $line = fgets($this->stdin);
        if ($line === false || $line === '') {
            return null;
        }

        $message = json_decode($line, true);
        if (!$message && json_last_error() !== JSON_ERROR_NONE) {
            throw new \Exception('Invalid JSON message');
        }

        return $message;
    }

    public function writeMessage(array $message): void
    {
        $json = json_encode($message);
        fwrite($this->stdout, $json . "\n");
        fflush($this->stdout);
    }

    public function close(): void
    {
        if ($this->stdin) fclose($this->stdin);
        if ($this->stdout) fclose($this->stdout);
    }
}
```

2. Create a transport factory:
```php
<?php

namespace App\Mcp\Transport;

class TransportFactory
{
    public static function create(string $type, $options = null): TransportInterface
    {
        return match($type) {
            'stdio' => new StdioTransport(),
            'sse' => new SseTransport($options),
            default => throw new \InvalidArgumentException("Unsupported transport type: $type")
        };
    }
}
```

## Step 4: Implement Core Server

Create the main MCP server class:

```php
<?php

namespace App\Mcp;

class Server implements ServerInterface
{
    private array $capabilities = [];
    private array $requestHandlers = [];
    private ?TransportInterface $transport = null;
    private bool $initialized = false;
    
    // ... implementation of interface methods ...
    // (See full Server.php implementation in previous message)
}
```

## Step 5: Create Base Procedure Class

```php
<?php

namespace App\Http\Procedures;

use Sajya\Server\Procedure;

class McpProcedure extends Procedure
{
    public static string $name = 'mcp';

    public function initialize()
    {
        return [
            'protocolVersion' => '2024-11-05',
            'implementation' => [
                'name' => 'laravel-mcp-server',
                'version' => '1.0.0'
            ],
            'capabilities' => [
                'resources' => [
                    'list' => true,
                    'read' => true
                ],
                'tools' => [
                    'list' => true,
                    'call' => true
                ]
            ]
        ];
    }

    // Add other required MCP methods
}
```

## Step 6: Create Console Command

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Mcp\Server;
use App\Mcp\Transport\StdioTransport;

class McpServer extends Command
{
    protected $signature = 'mcp:server';
    protected $description = 'Start MCP server';

    public function handle()
    {
        $server = new Server();
        $procedure = new McpProcedure();
        
        // Register handlers
        $server->setRequestHandler('initialize', [$procedure, 'initialize']);
        // Register other handlers...
        
        $transport = new StdioTransport();
        $server->connect($transport);
    }
}
```

## Step 7: Register Command

In `app/Console/Kernel.php`:
```php
protected $commands = [
    Commands\McpServer::class,
];
```

## Usage

Run the server:
```bash
php artisan mcp:server
```

## Extending the Server

To add new capabilities:

1. Create a new Resource:
```php
class MyResource implements ResourceInterface
{
    // Implement interface methods
}
```

2. Create a new Tool:
```php
class MyTool implements ToolInterface
{
    // Implement interface methods
}
```

3. Register them in your procedure class:
```php
class McpProcedure extends Procedure
{
    private MyResource $myResource;
    private MyTool $myTool;

    public function __construct()
    {
        $this->myResource = new MyResource();
        $this->myTool = new MyTool();
    }
    
    // Add corresponding methods to handle the new resource/tool
}
```

## Best Practices

1. Use dependency injection where possible
2. Implement proper error handling
3. Add logging for debugging
4. Write tests for your resources and tools
5. Follow Laravel's coding standards
6. Use type hints and return types
7. Document your code

This provides a foundation for building MCP servers in Laravel. You can extend it by adding more resources, tools, and capabilities as needed for your specific use case.
