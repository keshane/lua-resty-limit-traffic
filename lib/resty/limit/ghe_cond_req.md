```nginx
http {
    lua_shared_dict my_limit_req_store 100m;

    server {
        location / {
            access_by_lua_block {
                local limit_req = require "resty.limit.ghe_req"

                -- rate: 5000 requests per 3600s
                local lim, err = limit_req.new("api_limit_req_store", 5000, 3600)
                if not lim then
                    ngx.log(ngx.ERR, "failed to instantiate a resty.limit.req object: ", err)
                    return ngx.exit(500)
                end

                local headers = ngx.req.get_headers()

                local conditional = headers["If-None-Match"] or headers["If-Modified-Since"]
                -- use the Authorization header as the limiting key
                local key = headers["Authorization"] or "public"
                local remaining, reset = lim:incoming(key, true, conditional)


                if conditional and ngx.status == ngx.HTTP_NOT_MODIFIED then
                    remaining, reset = lim:outgoing(key, false, true)
                elseif conditional and ngx.status != ngx.HTTP_NOT_MODIFIED then
                    remaining, reset = lim:outgoing(key, true, true) 


                ngx.header["X-RateLimit-Limit"] = "5000"
                ngx.header["X-RateLimit-Reset"] = reset

                if remaining < 0 then
                    ngx.header["X-RateLimit-Remaining"] = 0
                    ngx.log(ngx.WARN, "rate limit exceeded")
                    return ngx.exit(403)
                else
                    ngx.header["X-RateLimit-Remaining"] = remaining
                end
            }
        }
    }
}
```
