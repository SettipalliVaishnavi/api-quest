#!/usr/bin/env bash
# Usage: bash test.sh [BASE_URL]   (default http://localhost:3000)
B=${1:-http://localhost:3000}; J='Content-Type: application/json'
c() { curl -s -o /dev/stderr -w "%{http_code}\n" "$@" 2>&1 | tr '\n' ' '; echo; }
echo "L1 ping:";        c $B/ping
echo "L2 echo:";        c -X POST $B/echo -H "$J" -d '{"hello":"world"}'
echo "L3 create:";      c -X POST $B/books -H "$J" -d '{"title":"1984","author":"George Orwell","year":1949}'
c -X POST $B/books -H "$J" -d '{"title":"Animal Farm","author":"George Orwell"}'
c -X POST $B/books -H "$J" -d '{"title":"Dune","author":"Frank Herbert"}'
echo "L3 list/get:";    c $B/books; c $B/books/1
echo "L4 update/del:";  c -X PUT $B/books/3 -H "$J" -d '{"title":"Dune Messiah"}'; c -X DELETE $B/books/3
TOKEN=$(curl -s -X POST $B/auth/token -H "$J" -d '{"username":"a","password":"b"}' | sed 's/.*"token":"\([^"]*\)".*/\1/')
echo "L5 no token (401):"; c $B/books
echo "L5 with token:";     c $B/books -H "Authorization: Bearer $TOKEN"
echo "L6 author/page:";    c "$B/books?author=George%20Orwell" -H "Authorization: Bearer $TOKEN"; c "$B/books?page=1&limit=1" -H "Authorization: Bearer $TOKEN"
echo "L7 invalid (400) / missing (404):"; c -X POST $B/books -H "$J" -d '{"title":""}'; c $B/books/999