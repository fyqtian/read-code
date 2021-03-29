### sqllite



```go
func main() {
   db, err := sql.Open("sqlite3", "/opt/test/sqllite/foo.db")
   fmt.Println(db, err)

   sql_table := `
      CREATE TABLE IF NOT EXISTS "userinfo" (
      "uid" INTEGER PRIMARY KEY AUTOINCREMENT,
      "username" VARCHAR(64) NULL,
      "departname" VARCHAR(64) NULL,
      "created" TIMESTAMP default (datetime('now', 'localtime'))
      );
`
   db.Exec(sql_table)

   stmt, err := db.Prepare("INSERT INTO userinfo(username, departname, created) values(?,?,?)")

   res, err := stmt.Exec("astaxie", "研发部门", "2012-12-09")

   fmt.Println(res)
}
```