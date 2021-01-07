jsp+servlet搭建在线投票问卷系统
=
 
``
如需源码请联系程序帮，QQ1022287044
``

开发环境准备
----
1. jdk1.8
2. tomcat8
3. mysql5.7
4. intellij IDEA

系统采用技术
----
1. jsp+ servlet mvc三层架构模式
2. jdbc
3. jQuery+ajax
4. ECharts 4.3.0

项目介绍
----

用户端


1. 用户端可以查看问卷列表并参与问卷调查

2. 查看个人参与的调查报告以及整个问卷情况

管理端


1. 问卷的新增和维护

2. 用户信息维护

项目设计
----
![设计流程图](/image/时序图/设计流程图/1.png)

![设计流程图](/image/时序图/设计流程图/6.png)




运行效果
----
1. 注册

![注册](/image/界面截图/注册.png)

2. 登录

![登录](/image/界面截图/登录.png)

3. 添加问卷

![添加问卷](/image/界面截图/添加问卷.png)

4. 问卷投票列表

![问卷投票列表](/image/界面截图/问卷投票.png)

5. 投票详情

![投票详情](/image/界面截图/问卷调查情况.png)

6. 用户列表

![](/image/界面截图/用户列表.png)

7. 数据库表

![](/image/数据库/数据库1.png)
![](/image/数据库/数据库2.png)

8. 代码结构截图

![代码结构截图](/image/数据库/项目结构.png)

关键代码:
-----
1. 添加问卷：
```diff
public void doGet(HttpServletRequest request, HttpServletResponse response) throws  IOException {
		String questionTitle = request.getParameter("questionTitle"); //问卷标题
		String qId = request.getParameter("qId"); //问卷id
		Integer ops =Integer.valueOf(request.getParameter("ops"));//具体几组
		String doType=request.getParameter("doType"); //操作类型

		User user=(User)request.getSession().getAttribute(User.SESSIONNAME);
		SubjectService subjectService=new SubjectServiceImpl();
		List<Subject> subjects=new ArrayList<>(); //问题列表
		Question question=new Question();
		question.setTitle(questionTitle);
		if("add".equalsIgnoreCase(doType)){ //如果修改添加id
			if(null!=user){//用户对象不为空，保存值
				question.setUserId(user.getId());
			}
		}else{
			question.setId(Integer.valueOf(qId));
		}
		String title="";
		for(int i=0;i<ops;i++){
			int number = Integer.parseInt(request.getParameter("number"+i));
			String[] os = request.getParameterValues("options"+i);
			String[] osIds = request.getParameterValues("optionsIds"+i);//选项id
			title=request.getParameter("title"+i);//问题标题
			String titleId=request.getParameter("titleId"+i);//问题id

			Subject subject = new Subject();
			subject.setOrderId(i);
			subject.setTitle(title);
			subject.setNumber(number);
			if(null!=titleId&&!"".equalsIgnoreCase(titleId)){//设置问题id
				subject.setId(Integer.valueOf(titleId));
			}
			for(int i1=0;i1<os.length;i1++){
				Option op = new Option();
				op.setContent(os[i1]);
				if("modify".equalsIgnoreCase(doType)){ //如果修改添加id
					op.setId(Integer.valueOf(osIds[i1]));
				}
				subject.getOptions().add(op);
			}
			try {
				long startTime=new Date().getTime();
				subject.setStartTime(startTime);
				subject.setEndTime(startTime+1*24*60*60*1000);
				subject.setMaster(user);
				subjects.add(subject);
			} catch (ReTryException e) {
				HttpSession session = request.getSession();
				session.setAttribute("subject", subject);
				session.setAttribute("message", e.getMessage());
				response.sendRedirect(request.getContextPath() + "/m/add");
			} catch (Exception ex) {
				throw new RuntimeException(ex);
			}
		}
		try{
			if("add".equalsIgnoreCase(doType)){//添加
				subjectService.add(question,subjects);
			}else{//修改
				subjectService.modify(question,subjects);
			}
		}catch (Exception e){
			e.printStackTrace();
		}

		if(null!=user&&user.getRole().equalsIgnoreCase("user")){
			response.sendRedirect(request.getContextPath() + "/list?action=myRelease");//用户查看自己的发布
		}else{
			response.sendRedirect(request.getContextPath() + "/list?action=index");
		}

	}
```
2. 修改问卷信息
```diff
public void doGet(HttpServletRequest request, HttpServletResponse response) {
		int id = Integer.parseInt(request.getParameter("id"));
		try {
			SubjectService subjectService = new SubjectServiceImpl();
			QuestionService questionService = new QuestionServiceImpl();
			SubjectQueryModel subjectQueryModel=new SubjectQueryModel();
			subjectQueryModel.setQuestionId(id);
			List<Subject> subject = subjectService.getSubjects(subjectQueryModel);//问题列表
			subjectQueryModel.setQuestionId(id);
			Question question=questionService.getQuestion(id);
			request.setAttribute("subjects", subject);
			request.setAttribute("ops", subject.size());//多少组
			request.setAttribute("question", question);
			request.getRequestDispatcher("/WEB-INF/pages/modify.jsp")
			       .forward(request, response);
		} catch (Exception e) {			
			throw new RuntimeException(e);
		}
	}
```

3. 问卷投票
```diff
    public void doGet(HttpServletRequest request, HttpServletResponse response)  {
    		try{
    			PrintWriter printWriter = response.getWriter();
    			int subjectId=Integer.parseInt(request.getParameter("subjectId"));
    			String id = request.getParameter("options");
    			String[] ids = id.split(",");
    			if(null==ids){ //未选中任何一项答案
    				printWriter.println(0);
    				return;
    			}else{
    				User user= (User)request.getSession().getAttribute(User.SESSIONNAME);
    				//如果登录用户，判断是否已经投过票
    				if(null!=user){
    					RecordQueryModel recordQueryModel=new RecordQueryModel();
    					recordQueryModel.getUser().setId(user.getId());
    					recordQueryModel.getSubject().setId(subjectId);
    					List<Record> list=recordService.getVotes(recordQueryModel);
    					if(list.size()>0){			//当前用户已经投过票
    						printWriter.println(2);
    						return;
    					}
    				}
    				List<Record> records = new ArrayList<>();
    				for(int i=0;i<ids.length;i++){
    					Record record = new Record();
    					record.getSubject().setId(subjectId);
    					record.getOption().setId(Integer.parseInt(ids[i]));
    					if(null!=user){
    						record.getUser().setId(user.getId());
    					}
    					records.add(record);
    				}
    				try {
    					recordService.vote(records); //保存问卷信息
    					printWriter.println(1);//参与成功
    					return;
    				} catch (ReTryException e) {
    					e.printStackTrace();
    					printWriter.println(3);//参与成功
    					return;
    				}catch(Exception ex){
    					ex.printStackTrace();
    					throw new RuntimeException(ex);
    				}
    			}
    		}	catch (Exception e){
    			e.printStackTrace();
    		}
    	}

```
 根据jdbc直连技术，编写数据库操作工具类，方便存储数据，代码如下：
```
public class DBUtils {
	String url = null;		//连接地址
	String username = null;		//数据库名
	String password = null;		//数据库密码
	String driverClass = null;	//连接驱动
	private static DBUtils db = new DBUtils();
	/**构建数据库连接参数*/
	private DBUtils() {
		try {
			url = "jdbc:mysql://localhost:3306/shopCartDb?useUnicode=true&characterEncoding=utf8";
			username = "root";
			password = "root123";
			driverClass = "com.mysql.jdbc.Driver";
			Class.forName(driverClass);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
	/**构建数据库连接对象*/
	public Connection getConnection(){
		Connection conn = null;
		try {
			conn = DriverManager.getConnection(url, username, password);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return conn;
	}
	public static DBUtils getInstance(){
		return db;
	}
}
```

4. 投票可视化代码

```diff
public void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		HttpSession session = request.getSession();
		try {
			int subjectId=Integer.parseInt(request.getParameter("subjectId")); //题目
			RecordService recordService=new RecordServiceImpl();
			 List<Record> list=recordService.geyVotes(subjectId);
			 if(list.size()>0){
				 session.setAttribute("list", list);
				 session.setAttribute("stitle", list.get(0).getSubject().getTitle());
				 List<String> opDatas=new ArrayList<>();
				 Map<String,Integer> map=new HashMap<>();
				 for(Record r:list){
					 if(null!=r.getOption()){
						 if(!opDatas.contains(r.getOption().getContent())){
							 opDatas.add(r.getOption().getContent());
							 map.put(r.getOption().getContent(),0);
						 }
						 map.put(r.getOption().getContent(),map.get(r.getOption().getContent())+1);
					 }

				 }
				 String opData="";
				 for(String o:opDatas){
					 opData=opData+"'"+o+"',";
				 }
				 if(opData.length()>0){
					 opData=opData.substring(0,opData.length()-1);
				 }
				 session.setAttribute("opData", opData);	//选项
				 String datavn="";
				 for (String key : map.keySet()) {
					 datavn=datavn+"{ value: "+map.get(key)+", name: '"+key+"' },";
				 }
				 if(datavn.length()>0){
					 datavn=datavn.substring(0,datavn.length()-1);
				 }
				 session.setAttribute("datavn", datavn);
				 request.getRequestDispatcher("/WEB-INF/pages/view.jsp").forward(request, response);
				 return;
			 }else{
			 	request.setAttribute("exception","暂无问卷结果！");
				 request.getRequestDispatcher("/WEB-INF/pages/500.jsp").forward(request, response);
				 return;
			 }
		} catch (ReTryException e) {
			request.getSession().setAttribute("message", e.getMessage());
			response.sendRedirect(request.getContextPath()+"/m/view");
		}catch(Exception ex){
			ex.printStackTrace();
		}
	}
```

 
