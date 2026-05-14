# lib/ip_block_middleware.rb
require 'net/http'
require 'json'
require 'uri'

class IpBlockMiddleware
  API_KEY = 'YOUR_API_KEY'
  BASE    = 'https://api.ip-block.com/v1'

  def initialize(app)
    @app = app
  end

  def api_reachable?
    uri  = URI("#{BASE}/ping")
    http = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl     = true
    http.open_timeout = 0.5
    http.read_timeout = 0.5
    res  = http.get(uri.path)
    JSON.parse(res.body)['status'] == 'ok'
  rescue
    false
  end

  def call(env)
    return @app.call(env) unless api_reachable? # fail open

    ip  = env['REMOTE_ADDR']
    uri = URI("#{BASE}/check")
    req = Net::HTTP::Post.new(uri, 'Content-Type' => 'application/json')
    req.body = {
      api_key:    API_KEY,
      ip:         ip,
      site_id:    'ABCDEFGHIJKL',
      user_agent: env['HTTP_USER_AGENT'].to_s,  # optional
      referrer:   env['HTTP_REFERER'].to_s      # optional
    }.to_json

    http     = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl = true
    res      = http.request(req)
    action   = JSON.parse(res.body)['action']

    return [302, { 'Location' => 'https://www.ip-block.com/blocked.php' }, []] if action == 'block'
    @app.call(env)
  end
end

# In config/application.rb (Rails) or config.ru (Sinatra):
# config.middleware.use IpBlockMiddleware