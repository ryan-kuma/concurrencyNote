#googleTest
###what makes a good test
1. tests should be independent and repeatable
2. tests should be well organized and relect the structure of the tested code
3. tests should be portabel and reusable
4. when tests fail, they should provide as much information about the problem as possible.
5. the testing framework should liberate test writers from housekeeping chores and let them focus on the test content.
6. teshts should be fast

###defferent definitions of the the terms Test, Test Case and Test Suite
- Tests use assertions to verify the tested code's behavior
- Test suite contains one or many tests

###different assertions
- ASSERT_* versions generate fatal filures when they fail and abort the current function
- EXPECT_* versions generate nonfatal failures and don't abort the current function


#concurrency
###why use concurrency
1. 为了逻辑划分的需要，不同的任务可以清楚的划分到不同的逻辑代码中
2. 为了性能的需要，可以并行运行在多核和多处理器电脑中，从而提升性能
### when not to use concurrency
当且仅当收益小于代价的时候 
###c++'s most vexing parse
```
TimeKeeper time_keeper(Timer());
```
该式子在c++中有两种解释，一种是定义一个TimeKeeper类型对象，参数为Timer类型的实例；另一种是声明一个返回值为TimeKeeper类型的函数,参数为返回类型为Timer的函数指针
c++标准把该语句解释为第二种情况
解决方法是使用闭包定义对象或者分两步
```
TimeKeeper time_keeper((Timer()));
//or
TimeKeeper time_keeper{Timer()};
//or
Timer timer();
TimeKeeper time_keeper(timer);
//or in c++11
TimeKeeper time_keeper([]{
	return Timer();
});
```
###合适的join和资源管理
由于程序随时可能因为exception而非正常退出，而忽略join和资源释放，所以
最理想的管理资源和线程的方式是通过RAII(Resource Acquisition Is Initialization)控制
```
class thread_guard
{
	thread& t;
public
	explicit thread_guard(thread &t_):t(t_){}
	~thread_guard()
	{
		if(t.joinable())
			t.join();
	}
	thread_guard(thread_guard const&)=delete;
	thread_guard& operator=(thread_guard const&)=delete;
};
struct func;
void f()
{
	int some_local_state = 0;
	func my_func(some_local_state);
	thread t(my_func);
	thread_guard g(t);

	do_something_in_current_thread();
}
```
#thread参数复制
thread构造实例时默认传参都是值传递,char const*指针会转换成string并复制到thread上下文空间
```
void f(int i, std::string const& s);
//此时oops函数可能会在buffer复制到thread context之前退出并销毁buffer
void oops(int some_param)
{
	char buffer[1024];
	sprintf(buffer, "%i", some_param);
	std::thread t(f, 3, buffer);//可以改成std::thread t(f, 3, std::string(buffer)),避免传指针
	t.detach();
}

```
需要用到引用传参时可以使用std::bind等包装参数
```
std::thread t(update_data_for_widget, w, std::ref(data));
```
#thread实例迁移
使用std::move()函数可将thread实例传递给另一个thread用于构造对象,利用这一点可改进之前的RAII线程对象管理方法，使用传值代替传引用方式
```
class scoped_thread
{
	std::thread t;
public:
	explicit scoped_thread(std::thread t_):t(std::move(t_))
	{
		if(!t.joinable())	
			throw std::logic_error("No thread");
	}
	~scoped_thread()
	{
		t.join();	
	}
	scoped_thread(scoped_thread const&)=delete;
	scoped_thread& operator=(scoped_thread const &)=delete;
};
struct func
{
	int& i;
	func(int& i_):i(i_){}
	void operator()()
	{
		for(unsigned j = 0; j < 10000000; j++)	
			do_something(i);
	}
};
void f()
{
	int some_local_state;
	scoped_thread t(std::thread(func(some_local_state)));
	do_something_in_current_thread();
}

```
#软件事务内存 (STM)
可用于多线程编程环境下避免静态条件，当需要修改数据结构时，将过程打包写入日志，再合适的时候一并提交处理
#mutex
可用mutex锁机制管理共享数据结构,但是存在死锁问题和保护数据太多太少的问题。
```
mutex some_mutex;
list<int> some_list;

void add_to_list(int new_value)
{
	lock_guard<mutex>guard(some_mutex);
	some_list.push_back(new_value);
}
bool list_contains(int value_to_find)
{
	lock_guard<mutex> guard(some_mutex);
	return find(some_list.begin(), some_list.end(), value_to_find)!=some_list.end();
}

```
要注意在保护数据时不要给函数留后门
```
class some_data
{
	int a;
	string b;
public:
	void do_something();
};

class data_wrapper
{
private:
	some_data data;
	mutex m;
public:
	template<typename Function>
	void process_data(Function func)
	{
		lock_guard<mutex> l(m);	//set mutex protect	data
		func(data);	//此时自定义函数中可能会给data留后门
	}

};

some_data* unprotected;
void malicious_function(some_data& protected_data)
{
	unprotected=&protected_data;
}
data_wrapper x;
void foo()
{
	x.process_data(malicious_function);
	unprotected->do_something();
}
```
尽量不要将被保护的变量通过指针和引用的方式传递到锁的外部。
