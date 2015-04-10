# Android平台上常用的定时器选择

Android平台定时器有两个：

>* Java原生的Timer
>* Android新引入的AlarmManager


##Java原生的Timer

```java

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		Timer timer = new Timer();
		timer.schedule(new MyTimerTask(), 2000);
	}

	static class MyTimerTask extends TimerTask {

		@Override
		public void run() {
			
			/* */
		}
	}
```

Timer实际上就是装装了一个Thread、一个TimerTask队列，这个TimerTask队列按照一定的方式排队执行。

但是，如果此时CUP进入了休眠状态，这个Thread就会因为失去了CPU而阻塞，导致我们的定时任务失效。

比如：我们设置一个任务5分钟后执行，可手机不到一分钟就锁屏进入休眠了，这时候我们这个任务就会执行失败。

##AlarmManager
AlarmManager是Android用来管理时钟的类，可以在手机休眠的时候正常运行，到达预设时间时，唤醒CPU来执行任务。

所以，如果我们用 AlarmManager 来定时执行任务，CPU 可以正常的休眠，只有在需要运行任务时醒来一段很短的时间。

##如何选择

我们需要根据当前场景来选择使用哪一个。

短时间的任务，可以通过Timer来实现，比如延时几百毫秒，几秒后执行某个任务。

对于长时间的定时任务，考虑到手机休眠导致的任务失效，改用AlarmManager来实现。例如，现在很多App的推送功能，实际上都是客户端定时发起查询，使用AlarmManager来每隔一小时、两小时执行一次的任务。

