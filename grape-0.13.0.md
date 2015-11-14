## 发送原生数据或无数据
一般，我们可以使用二进制格式发送原生数据。

	class API < Grape::API
	  get '/file' do
	    content_type 'application/octet-stream'
	    File.binread 'file.bin'
	  end
	end

你可以通过body显式的设置响应的内容。

	class API < Grape::API
	  get '/' do
	    content_type 'text/plain'
	    body 'Hello World'
	    # return value ignored
	  end
	end
使用body false会返回204没有内容或是没有没有内容类型。

你还可以设置响应文件类对象。注意：Rack返回响应之前将阅读你的整个枚举。如果你想响应流类型，请看streem.

	class FileStreamer
	  def initialize(file_path)
	    @file_path = file_path
	  end
	
	  def each(&blk)
	    File.open(@file_path, 'rb') do |file|
	      file.each(10, &blk)
	    end
	  end
	end
	
	class API < Grape::API
	  get '/' do
	    file FileStreamer.new('file.bin')
	  end
	end

如果你想将文件类响应为流，可以使用Rack::Chunchked来使用流。

	class API < Grape::API
	  get '/' do
	    stream FileStreamer.new('file.bin')
	  end
	end

## 授权

### 基本和摘要式身份认证
Grape内置基本和摘要式身份认证(在给定的块中执行代码).认证适用于当前的命名空间和任何子节点，而不包括父命名空间。

基本认证

	http_basic do |username, password|
	  # verify user's password here
	  { 'test' => 'password1' }[username] == password
	end

摘要认证

	http_digest({ realm: 'Test Api', opaque: 'app secret' }) do |username|
	  # lookup the user's password here
	  { 'user1' => 'password1' }[username]
	end

注册自定义中间件进行身份认证

Grape可以使用自定义的中间件进行身份认证。如果实现可以看看Rack::Auth::Basic或类似的实现。

注册一个中间件需要以下选项：

* label - 您的身份验证使用的名字
* MiddlewareClass - MiddlewareClass用于身份验证
* option_lookup_proc - 提供一个Proc作为参数给这个选项(返回值为数组，中间件参数)

例如：

	Grape::Middleware::Auth::Strategies.add(:my_auth, AuthMiddleware, ->(options) { [options[:realm]] } )
	
	
	auth :my_auth, { realm: 'Test Api'} do |credentials|
	  # lookup the user's password here
	  { 'user1' => 'password1' }[username]
	end

使用[warden-oauth2](https://github.com/opperator/warden-oauth2)或[rack-oauth2](https://github.com/nov/rack-oauth2)来支持OAuth2。

## 描述和检查的API
Grape的路由可以在运行时反射。这主要对生成的文档非常有用。

Grpae暴露API版本和编译的路由数组。每个路由都包括一个route_prefix、route_version、route_namespace, route_method, route_path和route_params。你可以通过route_setting添加自定义路由设置路由。

	class TwitterAPI < Grape::API
	  version 'v1'
	  desc "Includes custom settings."
	  route_setting :custom, key: 'value'
	  get do
	
	  end
	end

在运行时检查路由。

	TwitterAPI::versions # yields [ 'v1', 'v2' ]
	TwitterAPI::routes # yields an array of Grape::Route objects
	TwitterAPI::routes[0].route_version # => 'v1'
	TwitterAPI::routes[0].route_description # => 'Includes custom settings.'
	TwitterAPI::routes[0].route_settings[:custom] # => { key: 'value' }

## 当前路由和节点
可以从一个接口的调用中检索当前路由信息的信息。

	class MyAPI < Grape::API
	  desc "Returns a description of a parameter."
	  params do
	    requires :id, type: Integer, desc: "Identity."
	  end
	  get "params/:id" do
	    route.route_params[params[:id]] # yields the parameter description
	  end
	end

当前端点通过api块或env['api.endpoint']来响应内容。端点具有一些有趣的特性，比如source，它可以访问原代码块的执行。这对中间件构建记录器特别有用。

	class ApiLogger < Grape::Middleware::Base
	  def before
	    file = env['api.endpoint'].source.source_location[0]
	    line = env['api.endpoint'].source.source_location[1]
	    logger.debug "[api] #{file}:#{line}"
	  end
	end

## 之前和之后
可以指定一个块，在api每个调用之前或之后运行，使用before, after, before_validation 和 after_validation。

之前和之后的回调按下面的顺序执行：

1. before
2. before_validation
3. validations
4. after_validation
5. the API call
6. after

第4、5、6步只在验证通过的情况下执行：

例如，使用before:

	before do
	  header "X-Robots-Tag", "noindex"
	end

下面的块适用于当前命名空间每个api调用：

	class MyAPI < Grape::API
	  get '/' do
	    "root - #{@blah}"
	  end
	
	  namespace :foo do
	    before do
	      @blah = 'blah'
	    end
	
	    get '/' do
	      "root - foo - #{@blah}"
	    end
	
	    namespace :bar do
	      get '/' do
	        "root - foo - bar - #{@blah}"
	      end
	    end
	  end
	end

然后行为是：

	GET /           # 'root - '
	GET /foo        # 'root - foo - blah'
	GET /foo/bar    # 'root - foo - bar - blah'

在(你正在使用的或任何别名)命名空间参数时，使用before_validation或after_validation时也适用：

	class MyAPI < Grape::API
	  params do
	    requires :blah, type: Integer
	  end
	  resource ':blah' do
	    after_validation do
	      # if we reach this point validations will have passed
	      @blah = declared(params, include_missing: false)[:blah]
	    end
	
	    get '/' do
	      @blah.class
	    end
	  end
	end

行为如下：

	GET /123        # 'Fixnum'
	GET /foo        # 400 error - 'blah is invalid'

当一个回调被定义在指定的版本块中时，它只会在该块定义的路由中进行回调。

	class Test < Grape::API
	  resource :foo do
	    version 'v1', :using => :path do
	      before do
	        @output ||= 'v1-'
	      end
	      get '/' do
	        @output += 'hello'
	      end
	    end
	
	    version 'v2', :using => :path do
	      before do
	        @output ||= 'v2-'
	      end
	      get '/' do
	        @output += 'hello'
	      end
	    end
	  end
	end

这个api将呈现的行为为:

	GET /foo/v1       # 'v1-hello'
	GET /foo/v2       # 'v2-hello'

## 锚定
Grape在默认情况下锚定所有请求的路径，这意味着改请求URL应该从开始匹配，结束匹配，否则将返回404。然而，这又是不是你想要的，因为它并不可期的。这是由于Rack-mount在默认情况下锚请求从开始匹配到结尾，要么不匹配。Rails解决这个问题通过anchor:false在你的路由。在Grape可以使用这个选项：

例如当你的API需要url的一部分，例如：

	class TwitterAPI < Grape::API
	  namespace :statuses do
	    get '/(*:status)', anchor: false do
	
	    end
	  end
	end

这将匹配所有路径'/status/'。不过有一点需要注意，在params[:status]参数仅仅只有url的第一部分。幸运的是，还可以通过使用上述路径规范和使用Rack环境变量path_info来绕过，使用env['path_info']。这将捕获之后的'/statuese/'部分。

# 使用自定义中间件

## Rails中间件
请注意，如果你使用的Grape安装在Rails上，你不必使用Rails中间件，因为它以包含到您的中间件堆栈。你只需要执行helpers来访问特定的环境变量。

### 远程IP

默认情况下，你可以通过request.ip访问远程IP。这是由Rack实现的。有时候，你可以通过ActionDispatch::RemoteIp得到[Rails-style](http://stackoverflow.com/questions/10997005/whats-the-difference-between-request-remote-ip-and-request-ip-in-rails)的远程ip。

添加'actionpack'到你的Gemfile，并require 'action_dispatch/middleware/remote_ip.rb'。使用中间件API和暴露的client_ip助手。具体查看[文档](http://api.rubyonrails.org/classes/ActionDispatch/RemoteIp.html)。

	class API < Grape::API
	  use ActionDispatch::RemoteIp
	
	  helpers do
	    def client_ip
	      env["action_dispatch.remote_ip"].to_s
	    end
	  end
	
	  get :remote_ip do
	    { ip: client_ip }
	  end
	end

### 写入测试

#### 使用Rack的测试

在你的程序里使用rack-test定义在你的API。

##### RSpec

你可以使用RSpec通过HTTP请求检查响应来测试一个Grape API。

	require 'spec_helper'
	
	describe Twitter::API do
	  include Rack::Test::Methods
	
	  def app
	    Twitter::API
	  end
	
	  describe Twitter::API do
	    describe "GET /api/statuses/public_timeline" do
	      it "returns an empty array of statuses" do
	        get "/api/statuses/public_timeline"
	        expect(last_response.status).to eq(200)
	        expect(JSON.parse(last_response.body)).to eq []
	      end
	    end
	    describe "GET /api/statuses/:id" do
	      it "returns a status by id" do
	        status = Status.create!
	        get "/api/statuses/#{status.id}"
	        expect(last_response.body).to eq status.to_json
	      end
	    end
	  end
	end

##### Airborne

你可以与其他基于RSpec的框架，包括Airborne，它采用rack-test构建请求进行测试。

	require 'airborne'
	
	Airborne.configure do |config|
	  config.rack_app = Twitter::API
	end
	
	describe Twitter::API do
	  describe "GET /api/statuses/:id" do
	    it "returns a status by id" do
	      status = Status.create!
	      get "/api/statuses/#{status.id}"
	      expect_json(status.as_json)
	    end
	  end
	end

##### MiniTest

	require "test_helper"
	
	class Twitter::APITest < MiniTest::Test
	  include Rack::Test::Methods
	
	  def app
	    Twitter::API
	  end
	
	  def test_get_api_statuses_public_timeline_returns_an_empty_array_of_statuses
	    get "/api/statuses/public_timeline"
	    assert last_response.ok?
	    assert_equal [], JSON.parse(last_response.body)
	  end
	
	  def test_get_api_statuses_id_returns_a_status_by_id
	    status = Status.create!
	    get "/api/statuses/#{status.id}"
	    assert_equal status.to_json, last_response.body
	  end
	end

### 使用Rails编写测试

##### RSpec

	describe Twitter::API do
	  describe "GET /api/statuses/public_timeline" do
	    it "returns an empty array of statuses" do
	      get "/api/statuses/public_timeline"
	      expect(response.status).to eq(200)
	      expect(JSON.parse(response.body)).to eq []
	    end
	  end
	  describe "GET /api/statuses/:id" do
	    it "returns a status by id" do
	      status = Status.create!
	      get "/api/statuses/#{status.id}"
	      expect(response.body).to eq status.to_json
	    end
	  end
	end

在Rails中，HTTP请求测试将进入spec/requests组。你可能希望API代码进入app/api，您可以将匹配布局规范添加到spec/spec_helper.rb。

	RSpec.configure do |config|
	  config.include RSpec::Rails::RequestExampleGroup, type: :request, file_path: /spec\/api/
	end

##### MiniTest

	class Twitter::APITest < ActiveSupport::TestCase
	  include Rack::Test::Methods
	
	  def app
	    Rails.application
	  end
	
	  test "GET /api/statuses/public_timeline returns an empty array of statuses" do
	    get "/api/statuses/public_timeline"
	    assert last_response.ok?
	    assert_equal [], JSON.parse(last_response.body)
	  end
	
	  test "GET /api/statuses/:id returns a status by id" do
	    status = Status.create!
	    get "/api/statuses/#{status.id}"
	    assert_equal status.to_json, last_response.body
	  end
	end

#### 打桩助手

因为助手方法是混入节点中的，所以很难用短的或模拟测试它。Grpae::Endpoint.before_each方法可以让你在将每个请求之前运行端点定义行为提供帮助。

	describe 'an endpoint that needs helpers stubbed' do
	  before do
	    Grape::Endpoint.before_each do |endpoint|
	      allow(endpoint).to receive(:helper_name).and_return('desired_value')
	    end
	  end
	
	  after do
	    Grape::Endpoint.before_each nil
	  end
	
	  it 'should properly stub the helper' do
	    # ...
	  end
	end

## 在开发模式重新加载API更改

#### Rack程序重载
使用[grape-reload](https://github.com/AlexYankee/grape-reload)。

#### 使用Rails重载

添加api路径到config/application.rb中。

	# Auto-load API and its subdirectories
	config.paths.add File.join("app", "api"), glob: File.join("**", "*.rb")
	config.autoload_paths += Dir[Rails.root.join("app", "api", "*")]

创建config/initializers/reload_api.rb。

	if Rails.env.development?
	  ActiveSupport::Dependencies.explicitly_unloadable_constants << "Twitter::API"
	
	  api_files = Dir[Rails.root.join('app', 'api', '**', '*.rb')]
	  api_reloader = ActiveSupport::FileUpdateChecker.new(api_files) do
	    Rails.application.reload_routes!
	  end
	  ActionDispatch::Callbacks.to_prepare do
	    api_reloader.execute_if_updated
	  end
	end

更多信息请查看[StackOverflow #3282655](http://stackoverflow.com/questions/3282655/ruby-on-rails-3-reload-lib-directory-for-each-request/4368838#4368838)。

## 性能监控
### 活动支持仪表

Grape还内置了ActiveSupport::Notifications，提供简单钩指向应用的关键部分。

以下是目前支持的：

**endpoint_run.grape**

这是端点的主要执行，包括过滤器和渲染器。

* endpoint - 这个节点的实例。

**endpoint_render.grape**

这是断点主要内容的执行块。

* endpoint - 这个节点的实例。

**endpoint_run_filters.grape**

* endpoint - 端点的实例
* filters - 开始执行的过滤器
* type - 过滤器类型(before, before_validation, after_validation, after)

关于如何订阅这些事件的信息，请参阅[ActiveSupport::Notifications文档](http://api.rubyonrails.org/classes/ActiveSupport/Notifications.html)。

### 监控产品

Grape通过[newrelic-grape](https://github.com/flyerhzm/newrelic-grape)包集成NewRelic，通过[grape-librato](https://github.com/seanmoon/grape-librato)集成LIbrato Metrics。

## 对Grape项目贡献代码

Grape有数以百计的贡献者。我们鼓励你拉取源码，提出功能和讨论问题。

请查看[CONTRIBUTING](http://www.rubydoc.info/gems/grape/CONTRIBUTING.md)。

## 对Grape进行修剪

你能够只用几秒钟就在[Nitrous.IO](https://www.nitrous.io/?utm_source=github.com&utm_campaign=grape&utm_medium=hackonnitrous)网站上对Grape进行修剪。

[在Nitrous.IO上修建ruby-grape/grape](https://www.nitrous.io/hack_button?source=embed&runtime=rails&repo=intridea%2Fgrape&file_to_open=README.md)。

## 许可

MIT许可证。详情请参看许可证。

## 版权

Copyright (c) 2010-2015 Michael Bleigh, and Intridea, Inc.



