# lib/my_app_web/plugs/ip_block_plug.ex
defmodule MyAppWeb.IpBlockPlug do
  import Plug.Conn

  @api_key "YOUR_API_KEY"
  @base    "https://api.ip-block.com/v1"

  def init(opts), do: opts

  # Returns true if the API responds with %{"status" => "ok"}
  defp api_reachable? do
    case :httpc.request(:get,
           {'#{@base}/ping', []}, [{:timeout, 500}], []) do
      {:ok, {{_, 200, _}, _, body}} ->
        Jason.decode!(body)["status"] == "ok"
      _ -> false
    end
  end

  def call(conn, _opts) do
    if api_reachable?() do
      ip  = conn.remote_ip |> :inet.ntoa() |> to_string()
      ua  = get_req_header(conn, "user-agent") |> List.first("")  # optional
      ref = get_req_header(conn, "referer")    |> List.first("")  # optional
      payload = Jason.encode!(%{
        api_key:    @api_key,
        ip:         ip,
        site_id:    "ABCDEFGHIJKL",
        user_agent: ua,
        referrer:   ref
      })
      headers = [{'content-type', 'application/json'}]
      case :httpc.request(:post,
             {'#{@base}/check', headers, 'application/json', payload},
             [], []) do
        {:ok, {{_, 200, _}, _, body}} ->
          if Jason.decode!(body)["action"] == "block" do
            conn |> Phoenix.Controller.redirect(external: "https://www.ip-block.com/blocked.php") |> halt()
          else
            conn
          end
        _ -> conn
      end
    else
      conn  # fail open
    end
  end
end

# In router.ex: plug MyAppWeb.IpBlockPlug