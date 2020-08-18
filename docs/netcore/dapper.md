~~~c#
using (IDbConnection conn = new MySqlConnection(ConnectionString()))
{
    var result = conn.Execute<int>("sql",new { });
    onn.Close();
    return result;
}
~~~

##### 方法

- Execute

- Query

- QueryFirstOrDefault 返回查询到的第一个结果，否则返回默认值

- QueryFirst

- QuerySingle 返回查询到的第一个结果，否则返回异常

- QuerySingleOrDefault 返回一个查询结果，如果为空则返回默认值，如果结果多余1个则异常

- QueryMuliple 支持执行多个查询语句结果返回

  ~~~c#
  var gr = conn.QueryMultiple("",new {});
  response.Result = gr.Read<T>().ToList();
  response.TotalCount = int.Parse(gr.Read<Int64>().FirstOrDefault().ToString());
  ~~~

##### 参数

- 匿名

  ~~~c#
  new {}
  new[]{new {},new {}} //表示执行2次SQL，
  ~~~

- 动态类型

  ~~~c#
  DynamicParameters parameter = new DynamicParameters();
  parameter.Add("@Kind", InvoiceKind.WebInvoice, DbType.Int32,ParameterDirection.Input);
  ~~~

- 列表类型

  ~~~c#
  //Dapper允许您使用列表在IN子句中指定多个参数
  var sql = "SELECT * FROM Invoice WHERE Kind IN @Kind;";
  connection.Query<>(sql, new {Kind = new List<>{}}).ToList();
  ~~~

##### 事务

~~~c#
//封装dapper事务
public class MyTransaction : IDisposable
{
    public IDBConnection conn{get;set;}
    public IDBTransaction trans{get;set;}
    
    public MyTransaction(IDBConnection con){
        conn = con;
        conn.Open();
        trans = conn.BeginTransaction();
    }
    public void .Dispose(){
        trans.Dispose;
        conn.Dispose();
    }
    public void Commit(){
        trans.Commit();
        conn.Close();
    }
    public void RollBack(){
        trans.RollBack();
        conn.CLose();
    }
}
~~~

~~~c#
//使用
using(MyTransaction tbcontext = new MyTransaction())
{
    // 调用封装好的事务方法，IDbTransaction transaction
    connection.Execute(sb.ToString(), entityToInsert, transaction);//实际底层调用
    // 提交事务
    tbcontext.Commit();
}
~~~

