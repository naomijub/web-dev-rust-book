# Apêndice A - Requests

## Todo Server

### Signup

`POST http://localhost:4000/auth/signup`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf"
}
```

**Body**
```json
{
	"email": "my@email.com",
	"password": "My cr4zy p@ssw0rd My cr4zy p@ssw0rd"
}
```

### Login

`POST http://localhost:4000/auth/login`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf"
}
```

**Body**
```json
{
	"email": "my@email.com",
	"password": "My cr4zy p@ssw0rd My cr4zy p@ssw0rd"
}
```

### Logout

`DELETE http://localhost:4000/auth/logout`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf",
    "x-auth": "eyJhbGciOiJIUzI1NiIsImRhdGUiOiIyMDIwLTAyLTI4IDAxOjQxOjU2LjA2NjYxNTQwMCBVVEMiLCJ0eXAiOiJqd3QifQ.eyJlbWFpbCI6Im15QGVtYWlsLmNvbSIsImV4cGlyZXNfYXQiOiIyMDIwLTAyLTI5VDAxOjQxOjU2LjA2MzI2ODgwMCIsImlkIjoiZDdjNTk1MTItYjlhYS00NzBhLWEwNjUtZTAwYTYxMTcxYmE0In0.gIycarcQhbbcjvYIHDW_9fVgCFrFs1LjlJFMZGIm_kw"
}
```

**Body**
```json
{
	"email": "my@email.com"
}
```

### Create Todo

`POST http://localhost:4000/api/create`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf",
    "x-auth": "eyJhbGciOiJIUzI1NiIsImRhdGUiOiIyMDIwLTAyLTI4IDAxOjQxOjU2LjA2NjYxNTQwMCBVVEMiLCJ0eXAiOiJqd3QifQ.eyJlbWFpbCI6Im15QGVtYWlsLmNvbSIsImV4cGlyZXNfYXQiOiIyMDIwLTAyLTI5VDAxOjQxOjU2LjA2MzI2ODgwMCIsImlkIjoiZDdjNTk1MTItYjlhYS00NzBhLWEwNjUtZTAwYTYxMTcxYmE0In0.gIycarcQhbbcjvYIHDW_9fVgCFrFs1LjlJFMZGIm_kw"
}
```

**Body**
```json
{
	{
	"title": "title",
	"description": "descrition",
	"state": "Done",
	"owner": "90e700b0-2b9b-4c74-9285-f5fc94764995",
	"tasks": [
		{
			"is_done": true,
			"title": "blob"
			
		},
		{
			"is_done": false,
			"title": "blob2"
			
		}]
}
}
```

### Get All Todos

`GET http://localhost:4000/api/index`

**Headers**
```json
{
    "Content-Type": "application/json",
    "x-customer-id": "d8f98c2e-07df-4ed6-8645-7f0b25536fdf",
    "x-auth": "eyJhbGciOiJIUzI1NiIsImRhdGUiOiIyMDIwLTAyLTI4IDAxOjQxOjU2LjA2NjYxNTQwMCBVVEMiLCJ0eXAiOiJqd3QifQ.eyJlbWFpbCI6Im15QGVtYWlsLmNvbSIsImV4cGlyZXNfYXQiOiIyMDIwLTAyLTI5VDAxOjQxOjU2LjA2MzI2ODgwMCIsImlkIjoiZDdjNTk1MTItYjlhYS00NzBhLWEwNjUtZTAwYTYxMTcxYmE0In0.gIycarcQhbbcjvYIHDW_9fVgCFrFs1LjlJFMZGIm_kw"
}
```