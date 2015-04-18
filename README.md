# JavaWeb
Something about web
##购物商城之订单模块
###1、包和类
>**cn.itcast.bookstore.order**
>> **domain:**Order,OrderItem  
>>**dao:**OrderDao  
>> **service:**OrderService  
>>**web.servler:**OrderServlet
###2、生成订单功能
**Dao层**

	// 添加订单
	public void addOrder(Order order) {
		String sql = "insert into orders values(?,?,?,?,?,?)";
		Object[] params = { order.getOid(), order.getOrdertime(),
				order.getPrice(), order.getState(), order.getUser().getUid(),
				order.getAddress() };
		try {
			qr.update(sql, params);
		} catch (SQLException e) {
			throw new RuntimeException(e);
		}
	}
	// 添加订单条目
	public void addOrderItemList(List<OrderItem> orderItemList) {
		String sql = "insert into orderitem values(?,?,?,?,?)";
		// 创建二维数组
		Object[][] params = new Object[orderItemList.size()][];
		// 把多个OrderItem转换成多个一位数组，即一个二维数组
		// 循环遍历每个OrderItem让每个OrderItem初始化一个一位数组
		for (int i = 0; i < params.length; i++) {
			OrderItem item = orderItemList.get(i);
			params[i] = new Object[] { item.getIid(), item.getCount(),
					item.getSubtotal(), item.getOrder().getOid(),
					item.getBook().getBid() };
		}
		// 执行批处理
		try {
			qr.batch(sql, params);
		} catch (SQLException e) {
			throw new RuntimeException(e);
		}
	}
**service层**

	// 生成订单，
	public void creatOrder(Order order) {
		try {
			// 生成订单分两步，（并且放到同一个事务中）1、生成订单2、生成订单中的所有订单条目
			// 开启事务
			JdbcUtils.beginTransaction();
			// 生成订单
			od.addOrder(order);
			// 生成订单中的所有条目
			od.addOrderItemList(order.getOrderItemList());
			// 提交事务
			JdbcUtils.commitTransaction();
		} catch (Exception e) {
			try {
				JdbcUtils.rollbackTransaction();
			} catch (SQLException e1) {
			}
			throw new RuntimeException(e);
		}
	}
**servlet层**

	public String creatOrder(HttpServletRequest request,
			HttpServletResponse response) throws ServletException, IOException {
		request.setCharacterEncoding("utf-8");
		response.setContentType("text/html;charset=utf-8");
		// 获取Session中的车
		Cart cart = (Cart) request.getSession().getAttribute("cart");
		User user = (User) request.getSession().getAttribute("session_user");
		// 创建Order
		Order order = new Order();
		order.setOid(CommonUtils.uuid());
		order.setOrdertime(new Date());
		order.setPrice(cart.getTotal());
		order.setState(1);
		order.setUser(user);

		// 创建空orderItemList
		List<OrderItem> orderItemList=new ArrayList<OrderItem>();
		//向集合中添加订单条目
		for(CartItem cartItem:cart.getCartItems()){
			OrderItem orderItem=new OrderItem();
			orderItem.setIid(CommonUtils.uuid());
			orderItem.setCount(cartItem.getCount());
			orderItem.setSubtotal(cartItem.getSubtotal());
			orderItem.setOrder(order);
			orderItem.setBook(cartItem.getBook());
			orderItemList.add(orderItem);
		}
		//把素有条目设置给订单
		order.setOrderItemList(orderItemList);
		System.out.println(order);
		//调用Service完成条件
		os.creatOrder(order);
		//清空购物车
		cart.deleteAllCartItem();
		//保存order并转发到order/desc
		request.setAttribute("order", order);
		return "f:/jsps/order/desc.jsp";
	}
**domain层**
**Order**

	public class Order {
		private String oid;
		private Date ordertime;
		private double price;
		private int state;
		private User user;
		private String address;
		private List<OrderItem> orderItemList;// 本订单中所包含的所有条目
	
		public List<OrderItem> getOrderItemList() {
			return orderItemList;
		}
	
		public void setOrderItemList(List<OrderItem> orderItemList) {
			this.orderItemList = orderItemList;
		}
	
		public String getOid() {
			return oid;
		}
	
		public void setOid(String oid) {
			this.oid = oid;
		}
	
		public Date getOrdertime() {
			return ordertime;
		}
	
		public void setOrdertime(Date ordertime) {
			this.ordertime = ordertime;
		}
	
		public int getState() {
			return state;
		}
	
		public void setState(int state) {
			this.state = state;
		}
	
		public double getPrice() {
			return price;
		}
	
		public void setPrice(double price) {
			this.price = price;
		}
	
		public User getUser() {
			return user;
		}
	
		public void setUser(User user) {
			this.user = user;
		}
	
		public String getAddress() {
			return address;
		}
	
		public void setAddress(String address) {
			this.address = address;
		}
		@Override
		public String toString() {
			return "Order [oid=" + oid + ", ordertime=" + ordertime + ", price="
					+ price + ", state=" + state + ", user=" + user + ", address="
					+ address + ", orderItemList=" + orderItemList + "]";
		}
	}
**OrderItem**

	public class OrderItem {
		private String iid;
		private int count;
		private double subtotal;
		private Order order;
		private Book book;
		public String getIid() {
			return iid;
		}
		public void setIid(String iid) {
			this.iid = iid;
		}
		public int getCount() {
			return count;
		}
		public void setCount(int count) {
			this.count = count;
		}
		public double getSubtotal() {
			return subtotal;
		}
		public void setSubtotal(double subtotal) {
			this.subtotal = subtotal;
		}
		public Order getOrder() {
			return order;
		}
		public void setOrder(Order order) {
			this.order = order;
		}
		public Book getBook() {
			return book;
		}
		public void setBook(Book book) {
			this.book = book;
		}
	}
###3、我的订单功能
难度挺高  
思路：  
**Dao层**

	
	// 查询某人所有订单
	public List<Order> findByUid(String uid) {
		try {
			// 查询当前用户的所有订单（每个订单中没有订单条目）
			String sql = "select * from orders where uid=?";
			List<Order> orderList = qr.query(sql, new BeanListHandler<Order>(
					Order.class), uid);
			// 循环遍历每个order，为每个order加载订单条目
			for (Order order : orderList) {
				loadOrderItemList(order);
			}
			// 返回
			return orderList;

		} catch (SQLException e) {
			throw new RuntimeException(e);
		}
	}

	// 我的订单之为order加载它的所有的订单条目
	private void loadOrderItemList(Order order) throws SQLException {
		// 查询这个order的所有条目
		String sql = "select * from orderitem o,book b where o.bid=b.bid and oid=?";
		// 一条记录对应两张表，即对应两个对象，那么就使用MapListHandler
		List<Map<String, Object>> mapList = qr.query(sql, new MapListHandler(),
				order.getOid());
		// 返回的mapList中每个map对应一行记录，一行记录两个对象
		// 循环遍历每个map对象，把它转换成两个对象，添加到一个集合中
		List<OrderItem> orderItemList = new ArrayList<OrderItem>();
		for (Map<String, Object> map : mapList) {
			// 可以把map转换成两个对象，然后组合在一起
			OrderItem orderItem = toOrderItem(map);
			orderItemList.add(orderItem);
		}
		// 把得到的OrderItem 们给order对象
		order.setOrderItemList(orderItemList);
	}

	// 把一个map转换成两个对象，再组合在一起
	private OrderItem toOrderItem(Map<String, Object> map) {
		OrderItem orderItem = CommonUtils.toBean(map, OrderItem.class);
		Book book = CommonUtils.toBean(map, Book.class);
		// 组合在一起
		orderItem.setBook(book);
		return orderItem;
	}
**Service层**  

	public List<Order> myOrder(String uid) {
		return od.findByUid(uid);
	}
**Servlet层**  

	// 我的订单
	public String myOrder(HttpServletRequest request,
			HttpServletResponse response) throws ServletException, IOException {
		// 从Session中回去用户的uid
		// 使用uid调用service得到List(Order)
		// 保存转发
		User user = (User) request.getSession().getAttribute("session_user");
		List<Order> orderList = os.myOrder(user.getUid());
		request.setAttribute("orderList", orderList);
		return "f:/jsps/order/list.jsp";
	}

###4、加载订单功能

###5、确认收货功能

###6、付款之去银行

###7、付款之银行回调
