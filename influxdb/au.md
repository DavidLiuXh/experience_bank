## Influxdb授权验证流程分析
#### Qurey语句权限验证
1. 关于Influxdb的验证逻辑，我们可以先参考官方文档： [Authentication and authorization in InfluxDB](https://docs.influxdata.com/influxdb/v1.6/administration/authentication_and_authorization/)
1.1 在配置文件时，`[http] auth-enabled = true` 将开启授权验证
2. 我们从`appendHTTPDService`(cmd/influxdb/run/server.go)入手，在处理http request时要作授权验证，先要创建授权验证对象：
```
    srv.Handler.QueryAuthorizer = meta.NewQueryAuthorizer(s.MetaClient)
	srv.Handler.WriteAuthorizer = meta.NewWriteAuthorizer(s.MetaClient)
```
3. Query验证 `meta.QueryAuthorizer`(services/meta/data.go)
```
// QueryAuthorizer determines whether a user is authorized to execute a given query.
type QueryAuthorizer struct {
	Client *Client // MetaClient
}
```
4. 验证流程图：
![influxdb_authorizer.png](https://upload-images.jianshu.io/upload_images/2020390-b48ad0dcdc945e48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5. 通过上面这张图我们看到授权验证所有的接口都定义在 `type Authorizer interface`中
```
type Authorizer interface {
	// AuthorizeDatabase indicates whether the given Privilege is authorized on the database with the given name.
	AuthorizeDatabase(p influxql.Privilege, name string) bool

	// AuthorizeQuery returns an error if the query cannot be executed
	AuthorizeQuery(database string, query *influxql.Query) error

	// AuthorizeSeriesRead determines if a series is authorized for reading
	AuthorizeSeriesRead(database string, measurement []byte, tags models.Tags) bool

	// AuthorizeSeriesWrite determines if a series is authorized for writing
	AuthorizeSeriesWrite(database string, measurement []byte, tags models.Tags) bool
}
```
6. 首先看一下 `QueryAuthorizer.AuthorizeQuery, 以注释的方式给出说明
```
func (a *QueryAuthorizer) AuthorizeQuery(u User, query *influxql.Query, database string) error {
	// Special case if no users exist.
	if n := a.Client.UserCount(); n == 0 {
		// Ensure there is at least one statement.
		if len(query.Statements) > 0 {
			// First statement in the query must create a user with admin privilege.
            // 如果没有任何的用户，query的一个请求必须是Create Admin User
			cu, ok := query.Statements[0].(*influxql.CreateUserStatement)
			if ok && cu.Admin {
				return nil
			}
		}
        //Inlufxdb的验证流程规定，如果开启了授权验证，但还没有创建任何用户，则返回用户下面这个错误
		return &ErrAuthorize{
			Query:    query,
			Database: database,
			Message:  "create admin user first or disable authentication",
		}
	}

    //请求中没有user信息，则返回用户下面这个错误
	if u == nil {
		return &ErrAuthorize{
			Query:    query,
			Database: database,
			Message:  "no user provided",
		}
	}

    // 调用User.AuthorizeQuery继续验证
	return u.AuthorizeQuery(database, query)
}
```
7. 接着看一下`UserInfo.AuthorizeQuery`
```
func (u *UserInfo) AuthorizeQuery(database string, query *influxql.Query) error {
	// Admin privilege allows the user to execute all statements.
    // Admin用户有一切授权，不用再继续验证
	if u.Admin {
		return nil
	}

    // query里的每条statement的所需的所有privilege都需要逐一验证
	for _, stmt := range query.Statements {
		privs, err := stmt.RequiredPrivileges()
		for _, p := range privs {
			if p.Admin {
				// 走到这里说明当前用户一定不是admin用户，但当前statement又需要admin privilege, 则返回如下错
				return &ErrAuthorize{
					Query:    query,
					User:     u.Name,
					Database: database,
					Message:  fmt.Sprintf("statement '%s', requires admin privilege", stmt),
				}
			}

			db := p.Name
			if db == "" {
				db = database
			}
            // 调用u.AuthorizeDatabase进一步验证
			if !u.AuthorizeDatabase(p.Privilege, db) {
				return &ErrAuthorize{
					Query:    query,
					User:     u.Name,
					Database: database,
					Message:  fmt.Sprintf("statement '%s', requires %s on %s", stmt, p.Privilege.String(), db),
				}
			}
		}
	}
	return nil
}
```
8. 最后看一下`UserInfo.AuthorizeDatabase`和`UserInfo`的定义:
```
type UserInfo struct {
	// User's name.
	Name string

	// Hashed password.
	Hash string

	// Whether the user is an admin, i.e. allowed to do everything.
	Admin bool

	// Map of database name to granted privilege.
    // 针对不同的database的privilege都存在这个map中
	Privileges map[string]influxql.Privilege
}

 func (ui *UserInfo) AuthorizeDatabase(privilege influxql.Privilege, database string) bool {
	if ui.Admin || privilege == influxql.NoPrivileges {
		return true
	}
	p, ok := ui.Privileges[database]
    // 查Privileges这个map
	return ok && (p == privilege || p == influxql.AllPrivileges)
}
```

#### 写语句权限验证
1. `srv.Handler.WriteAuthorizer = meta.NewWriteAuthorizer(s.MetaClient)`，创建了写权限验证对象
2. 	具体实现 `WriteAuthorizer`(在 services/meta/write_authorizer.go中)， 实现比较简单， 最终也是调用`UserInfo.AuthorizeDatabase`来验证
```
// WriteAuthorizer determines whether a user is authorized to write to a given database.
type WriteAuthorizer struct {
	Client *Client
}

 // AuthorizeWrite returns nil if the user has permission to write to the database.
func (a WriteAuthorizer) AuthorizeWrite(username, database string) error {
	u, err := a.Client.User(username)
	if err != nil || u == nil || !u.AuthorizeDatabase(influxql.WritePrivilege, database) {
		return &ErrAuthorize{
			Database: database,
			Message:  fmt.Sprintf("%s not authorized to write to %s", username, database),
		}
	}
	return nil
}
```

#### Open权限验证
1. 不需要授权验证的情况，Influxdb实现了`openAuthorizer`, 它同样实现了`Authorizer interface`
```
 // OpenAuthorizer is the Authorizer used when authorization is disabled.
// It allows all operations.
type openAuthorizer struct{}

 // OpenAuthorizer can be shared by all goroutines.
var OpenAuthorizer = openAuthorizer{}
```