# Ruby Developer Guide

This guide covers how to integrate Ollama with Ruby applications, including Rails apps and testing with Minitest.

## Quick Start

### Basic HTTP Request

```ruby
require 'net/http'
require 'json'

# Generate completion
def generate_completion(model, prompt)
  uri = URI('http://localhost:11434/api/generate')
  
  request = Net::HTTP::Post.new(uri)
  request['Content-Type'] = 'application/json'
  request.body = {
    model: model,
    prompt: prompt,
    stream: false
  }.to_json
  
  response = Net::HTTP.start(uri.hostname, uri.port) do |http|
    http.request(request)
  end
  
  JSON.parse(response.body)
end

# Usage
result = generate_completion('mistral:7b', 'Why is the sky blue?')
puts result['response']
```

### Chat Completion

```ruby
def chat_completion(model, messages)
  uri = URI('http://localhost:11434/api/chat')
  
  request = Net::HTTP::Post.new(uri)
  request['Content-Type'] = 'application/json'
  request.body = {
    model: model,
    messages: messages,
    stream: false
  }.to_json
  
  response = Net::HTTP.start(uri.hostname, uri.port) do |http|
    http.request(request)
  end
  
  JSON.parse(response.body)
end

# Usage
messages = [
  { role: 'user', content: 'What is Ruby?' }
]
result = chat_completion('mistral:7b', messages)
puts result['message']['content']
```

## Inputs and Outputs

### Input Parameters

Common parameters for API requests:

```ruby
def ollama_request(endpoint, payload)
  uri = URI("http://localhost:11434/api/#{endpoint}")
  
  request = Net::HTTP::Post.new(uri)
  request['Content-Type'] = 'application/json'
  request.body = payload.to_json
  
  response = Net::HTTP.start(uri.hostname, uri.port) do |http|
    http.request(request)
  end
  
  JSON.parse(response.body)
end

# Generate with options
payload = {
  model: 'mistral:7b',
  prompt: 'Explain quantum computing',
  stream: false,
  options: {
    temperature: 0.7,
    top_p: 0.9,
    max_tokens: 500
  }
}

result = ollama_request('generate', payload)
```

### Output Structure

```ruby
# Generate endpoint returns:
{
  "model": "mistral:7b",
  "created_at": "2023-12-07T09:30:20.433906Z",
  "response": "The response text...",
  "done": true,
  "total_duration": 4935886791,
  "load_duration": 534986708,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 107345000,
  "eval_count": 237,
  "eval_duration": 4289432000
}

# Chat endpoint returns:
{
  "model": "mistral:7b",
  "created_at": "2023-12-07T09:30:20.433906Z",
  "message": {
    "role": "assistant",
    "content": "The response content..."
  },
  "done": true,
  "total_duration": 4935886791
}
```

## Forcing JSON Responses

### JSON Mode

```ruby
def get_structured_response(model, prompt)
  payload = {
    model: model,
    prompt: "#{prompt}. Respond using JSON format.",
    format: 'json',
    stream: false
  }
  
  result = ollama_request('generate', payload)
  JSON.parse(result['response'])
end

# Usage
prompt = "List 3 programming languages with their types"
structured_data = get_structured_response('mistral:7b', prompt)
# Returns parsed JSON object
```

### JSON Schema (Structured Outputs)

```ruby
def get_schema_response(model, prompt, schema)
  payload = {
    model: model,
    prompt: prompt,
    format: schema,
    stream: false
  }
  
  result = ollama_request('generate', payload)
  JSON.parse(result['response'])
end

# Define schema
person_schema = {
  type: 'object',
  properties: {
    name: { type: 'string' },
    age: { type: 'integer' },
    skills: {
      type: 'array',
      items: { type: 'string' }
    }
  },
  required: ['name', 'age']
}

# Usage
prompt = "Create a developer profile for Sarah, 28 years old"
person = get_schema_response('mistral:7b', prompt, person_schema)
```

## Response Caching

### Simple In-Memory Cache

```ruby
class OllamaCache
  def initialize
    @cache = {}
    @ttl = 3600 # 1 hour
  end
  
  def get(key)
    return nil unless @cache[key]
    
    entry = @cache[key]
    if Time.now - entry[:timestamp] > @ttl
      @cache.delete(key)
      return nil
    end
    
    entry[:data]
  end
  
  def set(key, data)
    @cache[key] = {
      data: data,
      timestamp: Time.now
    }
  end
  
  def cache_key(model, prompt, options = {})
    Digest::SHA256.hexdigest("#{model}:#{prompt}:#{options.to_json}")
  end
end

# Usage with caching
class OllamaClient
  def initialize
    @cache = OllamaCache.new
  end
  
  def generate_with_cache(model, prompt, options = {})
    cache_key = @cache.cache_key(model, prompt, options)
    
    # Check cache first
    cached_result = @cache.get(cache_key)
    return cached_result if cached_result
    
    # Make API call
    payload = {
      model: model,
      prompt: prompt,
      stream: false,
      options: options
    }
    
    result = ollama_request('generate', payload)
    
    # Cache the result
    @cache.set(cache_key, result)
    result
  end
end
```

### Redis Cache

```ruby
require 'redis'

class OllamaRedisCache
  def initialize(redis_url = 'redis://localhost:6379')
    @redis = Redis.new(url: redis_url)
    @ttl = 3600
  end
  
  def get(key)
    cached = @redis.get(key)
    cached ? JSON.parse(cached) : nil
  end
  
  def set(key, data)
    @redis.setex(key, @ttl, data.to_json)
  end
  
  def cache_key(model, prompt, options = {})
    "ollama:#{Digest::SHA256.hexdigest("#{model}:#{prompt}:#{options.to_json}")}"
  end
end
```

## Testing with Minitest

### Basic Mock Setup

```ruby
require 'minitest/autorun'
require 'webmock/minitest'

class OllamaTest < Minitest::Test
  def setup
    WebMock.enable!
    # Mock the Ollama server
    @ollama_url = 'http://localhost:11434'
  end
  
  def teardown
    WebMock.reset!
  end
  
  def test_generate_completion
    # Mock the API response
    mock_response = {
      model: 'mistral:7b',
      response: 'The sky is blue due to Rayleigh scattering.',
      done: true,
      total_duration: 1000000000
    }
    
    stub_request(:post, "#{@ollama_url}/api/generate")
      .with(
        body: {
          model: 'mistral:7b',
          prompt: 'Why is the sky blue?',
          stream: false
        }.to_json,
        headers: { 'Content-Type' => 'application/json' }
      )
      .to_return(
        status: 200,
        body: mock_response.to_json,
        headers: { 'Content-Type' => 'application/json' }
      )
    
    # Test your code
    result = generate_completion('mistral:7b', 'Why is the sky blue?')
    
    assert_equal 'The sky is blue due to Rayleigh scattering.', result['response']
    assert_equal true, result['done']
  end
end
```

### Rails Integration Tests

```ruby
# test/integration/ai_features_test.rb
require 'test_helper'

class AiFeaturesTest < ActionDispatch::IntegrationTest
  def setup
    WebMock.enable!
    @ollama_url = 'http://localhost:11434'
  end
  
  def teardown
    WebMock.reset!
  end
  
  def test_ai_powered_feature
    # Mock Ollama response
    mock_ai_response = {
      model: 'mistral:7b',
      message: {
        role: 'assistant',
        content: 'This is a helpful AI response'
      },
      done: true
    }
    
    stub_request(:post, "#{@ollama_url}/api/chat")
      .to_return(
        status: 200,
        body: mock_ai_response.to_json,
        headers: { 'Content-Type' => 'application/json' }
      )
    
    post '/ai/chat', params: {
      message: 'Hello AI'
    }
    
    assert_response :success
    response_data = JSON.parse(response.body)
    assert_equal 'This is a helpful AI response', response_data['ai_response']
  end
end
```

### Service Object Testing

```ruby
# app/services/ai_service.rb
class AiService
  OLLAMA_URL = 'http://localhost:11434'
  
  def self.chat(messages, model: 'mistral:7b')
    uri = URI("#{OLLAMA_URL}/api/chat")
    
    request = Net::HTTP::Post.new(uri)
    request['Content-Type'] = 'application/json'
    request.body = {
      model: model,
      messages: messages,
      stream: false
    }.to_json
    
    response = Net::HTTP.start(uri.hostname, uri.port) do |http|
      http.request(request)
    end
    
    result = JSON.parse(response.body)
    result['message']['content']
  end
end

# test/services/ai_service_test.rb
require 'test_helper'

class AiServiceTest < ActiveSupport::TestCase
  def setup
    WebMock.enable!
  end
  
  def teardown
    WebMock.reset!
  end
  
  def test_chat_returns_ai_response
    mock_response = {
      model: 'mistral:7b',
      message: {
        role: 'assistant',
        content: 'Hello! How can I help you?'
      },
      done: true
    }
    
    stub_request(:post, "#{AiService::OLLAMA_URL}/api/chat")
      .with(
        body: hash_including(
          model: 'mistral:7b',
          messages: [{ role: 'user', content: 'Hello' }]
        )
      )
      .to_return(
        status: 200,
        body: mock_response.to_json
      )
    
    messages = [{ role: 'user', content: 'Hello' }]
    result = AiService.chat(messages)
    
    assert_equal 'Hello! How can I help you?', result
  end
  
  def test_chat_handles_errors
    stub_request(:post, "#{AiService::OLLAMA_URL}/api/chat")
      .to_return(status: 500)
    
    messages = [{ role: 'user', content: 'Hello' }]
    
    assert_raises(StandardError) do
      AiService.chat(messages)
    end
  end
end
```

### Test Fixtures

```ruby
# test/fixtures/ollama_responses.rb
module OllamaResponses
  def self.generate_response(text = 'Sample AI response')
    {
      model: 'mistral:7b',
      created_at: Time.current.iso8601,
      response: text,
      done: true,
      total_duration: 1000000000,
      load_duration: 50000000,
      prompt_eval_count: 10,
      prompt_eval_duration: 100000000,
      eval_count: 25,
      eval_duration: 500000000
    }
  end
  
  def self.chat_response(content = 'Sample chat response')
    {
      model: 'mistral:7b',
      created_at: Time.current.iso8601,
      message: {
        role: 'assistant',
        content: content
      },
      done: true,
      total_duration: 1000000000
    }
  end
  
  def self.json_response(data = { key: 'value' })
    {
      model: 'mistral:7b',
      created_at: Time.current.iso8601,
      response: data.to_json,
      done: true,
      total_duration: 1000000000
    }
  end
end

# Usage in tests
def test_with_fixture
  stub_request(:post, "http://localhost:11434/api/generate")
    .to_return(
      status: 200,
      body: OllamaResponses.generate_response('Custom response').to_json
    )
  
  # Your test code here
end
```

## Production Considerations

### Connection Pooling

```ruby
require 'net/http/persistent'

class OllamaClient
  def initialize(base_url = 'http://localhost:11434')
    @base_url = base_url
    @http = Net::HTTP::Persistent.new('ollama-client')
    @http.idle_timeout = 30
    @http.keep_alive = 30
  end
  
  def request(endpoint, payload)
    uri = URI("#{@base_url}/api/#{endpoint}")
    
    request = Net::HTTP::Post.new(uri)
    request['Content-Type'] = 'application/json'
    request.body = payload.to_json
    
    response = @http.request(uri, request)
    JSON.parse(response.body)
  ensure
    # Connection will be reused due to persistent connection
  end
  
  def close
    @http.shutdown
  end
end
```

### Timeout Handling

```ruby
class OllamaClient
  def initialize(base_url = 'http://localhost:11434', timeout: 30)
    @base_url = base_url
    @timeout = timeout
  end
  
  def request_with_timeout(endpoint, payload)
    uri = URI("#{@base_url}/api/#{endpoint}")
    
    Net::HTTP.start(uri.hostname, uri.port, 
                    open_timeout: @timeout, 
                    read_timeout: @timeout) do |http|
      request = Net::HTTP::Post.new(uri)
      request['Content-Type'] = 'application/json'
      request.body = payload.to_json
      
      response = http.request(request)
      JSON.parse(response.body)
    end
  rescue Net::TimeoutError => e
    raise StandardError, "Ollama request timed out: #{e.message}"
  end
end
```

## Community Ruby Gem

For more advanced Ruby integration, consider using the community-maintained gem:

```ruby
# Gemfile
gem 'ollama-ai'

# Usage
require 'ollama-ai'

client = Ollama.new(
  credentials: {
    address: 'http://localhost:11434'
  },
  options: { server_sent_events: true }
)

result = client.generate(
  { model: 'mistral',
    prompt: 'Hi!' }
)
```

The gem provides additional features like streaming responses, better error handling, and more Ruby-idiomatic APIs. See [ollama-ai](https://github.com/gbaptista/ollama-ai) for full documentation.

## Additional Resources

- [Ollama API Documentation](./api.md)
- [Model Library](https://ollama.com/library)
- [WebMock Documentation](https://github.com/bblimke/webmock) for testing HTTP requests
- [Minitest Documentation](https://github.com/minitest/minitest)