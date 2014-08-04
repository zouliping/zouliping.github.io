关于Jena的简介在很多博客中都能看到，例如对Jena的简单理解和一个例子，使用Jena将本体存入MySQL，Jena进阶等。在入门的时候，看这些文章总是很疑惑，对于存入数据库后的操作一无所知。因此，在做项目之余，把使用到的一些方法记录下来，以供将来查阅。
###将本体持久化到数据库
首先，还是不能免俗，说说如何将本体持久化到数据库的方法。总体的操作流程是将本体从owl文件中读出，建立一个数据库的连接，将Model存入数据库。这个过程的操作比较简单，见如下代码。其中需要注意的一点是：在这里的url设置是不正确的，这样设置的结果将导致中文乱码，正确的设置方法参见本文最后提到的问题解决方案。

{ highlight java }
private final static String driver = "com.mysql.jdbc.Driver";
private final static String url = "jdbc:mysql://localhost/dbname";
private final static String db = "MySQL";
private final static String user = "your user name";
private final static String pwd = "your password";

public static void main(String[] args) {
	try {
			IDBConnection con = getConnection(url, user, pwd, db);

			Class.forName(driver);

			String path = "your owl file path";
			createModel(con, "model name", path);

			con.close();
		} catch (SQLException e) {
			e.printStackTrace();
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
	}

/**
* get db connection
* 
* @param dbUrl
* @param dbUser
* @param dbPwd
* @param dbName
* @return
*/
public static DBConnection getConnection(String dbUrl, String dbUser,
			String dbPwd, String dbName) {
		return new DBConnection(dbUrl, dbUser, dbPwd, dbName);
	}

/**
* read owl file, create the ontModel, and store in db
* 
* @param conn
* @param name
* @param filePath
* @return
*/
public static OntModel createModel(IDBConnection conn, String name,
			String filePath) {
		ModelMaker maker = ModelFactory.createModelRDBMaker(conn);
		Model model = maker.createModel(name);

		try {
			File file = new File(filePath);
			FileInputStream fis = new FileInputStream(file);
			InputStreamReader isr = new InputStreamReader(fis, "UTF-8");
			model.read(isr, null);

			isr.close();
			fis.close();

			model.commit();
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		} catch (UnsupportedEncodingException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}

		OntModelSpec spec = new OntModelSpec(OntModelSpec.OWL_MEM);
		return ModelFactory.createOntologyModel(spec, model);
}
{ endhighlight }