# PullToRefreshView
简易的下拉刷新控件
##前言
下拉刷新控件，想必大家都用过，现在网上下拉刷新的库很多像PullToRefresh（https://github.com/chrisbanes/Android-PullToRefresh）等等，类似的控件有很多，开源是个好东西，能使我们节省时间，但一味的去使用，不去了解实现的原理，只会让我们程序员成为代码的搬运工。

下图是我们今天要实现的效果：


![下拉刷新](http://img.blog.csdn.net/20160308194631369)



##刷新原理

整个刷新控件包括两个部分：刷新的头部+ListView。
在实现我们的下拉刷新控件时，需要注意一下几点：

 1. 一开始进入我们的列表页时，刷新的头部是需要隐藏的。
 2. 滑动ListView时，是什么时候该下拉刷新，什么时候又是纯粹的往上滑动。也就是说只有在ListView滑动到顶部时，才是决定是否下拉。
 3. ListView滑动到顶部时，往下拉一半的头部View时松手，这时应该判定为刷新行为失败，当整个头部被下拉时松手判定为刷新成功。
 
 基于以上说明，我们先为我们的控件增加头部的View:

```
package com.example.pulltorefreshlistview.view;

import android.content.Context;
import android.util.AttributeSet;
import android.view.Gravity;
import android.widget.FrameLayout;
import android.widget.TextView;

/**
 * Created by glh on 2016/3/1.
 */
public class PullHeaderView extends FrameLayout {

	private Context mContext;
	private TextView mHeaderText;

	public PullHeaderView(Context context) {
		this(context, null);
	}

	public PullHeaderView(Context context, AttributeSet attrs) {
		this(context, attrs, 0);
	}

	public PullHeaderView(Context context, AttributeSet attrs, int defStyleAttr) {
		super(context, attrs, defStyleAttr);
		mContext = context;
		init();
	}

	private void init() {
		mHeaderText = new TextView(mContext);
		mHeaderText.setTextSize(20);
		mHeaderText.setPadding(20, 20, 20, 20);
		mHeaderText.setGravity(Gravity.CENTER);
		addView(mHeaderText);

	}

	protected void start() {
		mHeaderText.setText("下拉刷新");
	}

	protected void loading() {
		mHeaderText.setText("正在刷新...");
	}

	protected void end() {
		mHeaderText.setText("刷新结束");
	}

}

```

头部的视图很简单，只显示一个TextView，用于显示刷新的状态。

刷新的控件PullToRefreshView继承LinearLayout，先进行相应的初始化：

```
public class PullToRefreshView extends LinearLayout implements
		View.OnTouchListener {

    /**
	 * 下拉头的View
	 */
	private PullHeaderView header;
	/**
	 * 下拉控件布局参数
	 */
	private MarginLayoutParams headerLayoutParams;
	
	/**
	 * 下拉状态
	 */
	public static final int STATUS_PULL_TO_REFRESH = 0;

	/**
	 * 释放准备刷新状态
	 */
	public static final int STATUS_RELEASE_TO_REFRESH = 1;

	/**
	 * 正在刷新状态
	 */
	public static final int STATUS_REFRESHING = 2;

	/**
	 * 刷新完成状态
	 */

     /**
	 * 在被判定为滚动之前用户手指可以移动的最大值
	 */
	private int touchSlop;

	public PullToRefreshView(Context context) {
		this(context, null);
	}

	public PullToRefreshView(Context context, AttributeSet attrs) {
		this(context, attrs, 0);
	}

	public PullToRefreshView(Context context, AttributeSet attrs,
			int defStyleAttr) {
		super(context, attrs, defStyleAttr);
		setOrientation(VERTICAL);
		touchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
		initView(context, attrs);
	}

	private void initView(Context context, AttributeSet attrs) {
		header = new PullHeaderView(context, attrs);
		header.start();
		addView(header, 0);
	}

}
```

上面的代码很简单，定义几个刷新的状态：

 1. STATUS_PULL_TO_REFRESH   下拉状态
 2. STATUS_RELEASE_TO_REFRESH 释放准备刷新状态
 3. STATUS_REFRESHING 正在刷新状态
 4. STATUS_REFRESH_FINISHED 刷新完成状态
 

> 用于保存我们在滑动时的ListView状态，initView方法也很容易理解，创建我们的头部视图，并进行初始化，最后添加到我们的PullToRefreshView(继承与LinearLayout)的第一个位置上。构造器里面也只是获取当前屏幕滑动的最小距离。

这个时候运行程序，效果是这样的：

![未隐藏头部时效果](http://img.blog.csdn.net/20160308201343161)

查看上面的图片，我们会看到头部视图被完全显示出来了，这不是我们想要的效果，往上翻看之前说过的注意点1，知道我们要的效果是，一进去头部时被隐藏的，那如何隐藏呢，这里面就需要重写onLayout方法。

> 我们知道View的工作流程主要包括measure、layout、draw这三大流程，也就是测量、布局和绘制，measure用于确定View的测量宽和高，layout确定View的最终确定宽和高以及位置，而draw将View绘制到屏幕，在这里我们只实现onLayout方法，用于确定我们的头部视图的位置。

```
	/**
	 * 用于控制onLayout中的初始化只需加载一次
	 */
	private boolean once;
	/**
	 * 下拉控件高度
	 */
	private int hideHeaderHeight;
	/**
	 * 需要去下拉刷新的ListView
	 */
	private ListView listView;
	@Override
	protected void onLayout(boolean changed, int l, int t, int r, int b) {
		super.onLayout(changed, l, t, r, b);
		if (changed && !once) {
			hideHeaderHeight = -header.getHeight();
			headerLayoutParams = (MarginLayoutParams) header.getLayoutParams();
			headerLayoutParams.topMargin = hideHeaderHeight;
			listView = (ListView) getChildAt(1);
			listView.setOnTouchListener(this);
			once = true;
		}
	}
	
	@Override
	public boolean onTouch(View v, MotionEvent event) {
		// TODO Auto-generated method stub
		return false;
	}
```

onLayout方法中主要做了以下几步：

 1. 获取头部header的高度
 2. 获取头部header的布局参数MarginLayoutParams
 3. 为header设置布局参数（-header.getHeight()），进行隐藏。
 4. 拿到我们的ListView,并给它设置触摸事件。


运行程序，显示如下：

![隐藏头部视图](http://img.blog.csdn.net/20160308202634162)

到这里为止，已经将头部视图给隐藏了，接下来就是我们的滑动监听了，实现onTouch方法：

```
	/**
	 * 当前是否可以下拉，只有ListView滚动到头的时候才允许下拉
	 */
	private boolean ableToPull;
	
	/**
	 * 当前状态
	 */
	private int currentStatus = STATUS_REFRESH_FINISHED;

	/**
	 * 手指按下时屏幕纵坐标
	 */
	private float preDownY;

	/**
	 * 下拉拖动的黏性比率
	 */
	private static final float STICK_RATIO = .65f;

	/**
	 * 下拉刷新的回调接口
	 */
	private RefreshListener mListener;
	
	private int tempHeaderTopMargin;

    @Override
	public boolean onTouch(View v, MotionEvent event) {
		checkAblePull(event);
		if (!ableToPull) {
			/**
			 * 只有ListView滚动到头的时候才允许下拉
			 */
			return false;
		}
		switch (event.getAction()) {
		case MotionEvent.ACTION_DOWN:
			preDownY = event.getRawY();
			break;
		case MotionEvent.ACTION_MOVE:
			float currY = event.getRawY();
			float distance = currY - preDownY;
			float offsetY = distance * STICK_RATIO;
			if (distance <= 0
					&& headerLayoutParams.topMargin <= hideHeaderHeight) {
				return false;
			}
			if (distance < touchSlop) {
				return false;
			}

			header.start();
			if (headerLayoutParams.topMargin > 0) {
				currentStatus = STATUS_RELEASE_TO_REFRESH;
			} else {
				currentStatus = STATUS_PULL_TO_REFRESH;
			}

			// 通过偏移下拉头的topMargin值，来实现下拉效果
			setHeaderTopMarign((int) (offsetY + hideHeaderHeight));

			break;
		case MotionEvent.ACTION_UP:
			if (currentStatus == STATUS_PULL_TO_REFRESH) {
				/**
				 * 下拉状态
				 */
				rollbackHeader();
			}
			if (currentStatus == STATUS_RELEASE_TO_REFRESH) {
				/**
				 * 头部向下拉时，松手后的逻辑
				 */
				scrollBackHeader();
			}
			break;
		}
		if (currentStatus == STATUS_PULL_TO_REFRESH
				|| currentStatus == STATUS_RELEASE_TO_REFRESH) {
			// 让ListView失去焦点, 不可被点击
			disableListView();
			return true;
		}
		return false;
	}
	
	/**
	 * 根据当前ListView的滚动状态来设定 {@link #ableToPull}
	 * 的值，每次都需要在onTouch中第一个执行，这样可以判断出当前应该是滚动ListView，还是应该进行下拉。
	 * 
	 * @param event
	 */
	private void checkAblePull(MotionEvent event) {
		View firstChild = listView.getChildAt(0);
		if (firstChild != null) {
			int firstVisiblePos = listView.getFirstVisiblePosition();
			if (firstVisiblePos == 0 && firstChild.getTop() == 0) {
				/**
				 * 如果首个元素的上边缘，距离父布局值为0，就说明ListView滚动到了最顶部，此时应该允许下拉刷新
				 */
				if (!ableToPull) {
					preDownY = event.getRawY();
				}
				ableToPull = true;
			} else { // 反之
				if (headerLayoutParams.topMargin != hideHeaderHeight) {
					setHeaderTopMarign(hideHeaderHeight);
				}
				ableToPull = false;
			}
		} else {
			/**
			 * 如果ListView中没有元素，也应该允许下拉刷新
			 */
			ableToPull = true;
		}
	}


	/**
	 * 回滚到头部刷新控件的高度，并触发后台刷新任务
	 */
	private void scrollBackHeader() {
		/**
		 * 当我们将头部布局一直往下来松手收，使它能平滑的回到顶部。
		 */
		ValueAnimator rbToHeaderAnimator = ValueAnimator.ofInt(
				headerLayoutParams.topMargin, 0);
		long duration = (long) (headerLayoutParams.topMargin * 1.1f) >= 0 ? (long) (headerLayoutParams.topMargin * 1.1f)
				: 0;
		rbToHeaderAnimator.setDuration(duration);
		rbToHeaderAnimator
				.setInterpolator(new AccelerateDecelerateInterpolator());
		rbToHeaderAnimator
				.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
					@Override
					public void onAnimationUpdate(ValueAnimator animation) {
						int marginValue = Integer.parseInt(animation
								.getAnimatedValue().toString());
						setHeaderTopMarign(marginValue);
					}
				});
		rbToHeaderAnimator.addListener(new AnimatorListenerAdapter() {
			@Override
			public void onAnimationEnd(Animator animation) {
				currentStatus = STATUS_REFRESHING;
				header.loading();
				Executors.newSingleThreadExecutor().submit(new Runnable() {
					@Override
					public void run() {
						if (mListener != null) {
							mListener.onRefreshing();
						}
					}
				});
			}
		});
		rbToHeaderAnimator.start();
	}

	/**
	 * 给header设置topMargin参数
	 * 
	 * @param margin
	 */
	private void setHeaderTopMarign(int margin) {
		headerLayoutParams.topMargin = margin;
		header.setLayoutParams(headerLayoutParams);
	}

	/**
	 * 禁用ListView，让其失去焦点不可接受点击
	 */
	private void disableListView() {
		listView.setPressed(false);
		listView.setFocusable(false);
		listView.setFocusableInTouchMode(false);
	}


	/**
	 * 给下拉刷新控件注册一个监听器。
	 * 
	 * @param listener
	 *            监听器的实现。
	 */
	public void setOnRefreshListener(RefreshListener listener) {
		mListener = listener;
	}

	/**
	 * 当所有的刷新逻辑完成后，记录调用一下，否则你的ListView将一直处于正在刷新状态。
	 */
	public void finishRefreshing() {
		header.end();
		/**
		 * 回滚下拉刷新头部控件
		 */
		rollbackHeader();
	}



	/**
	 * 回滚下拉刷新头部控件
	 */
	private void rollbackHeader() {
		tempHeaderTopMargin = headerLayoutParams.topMargin;
		ValueAnimator rbAnimator = ValueAnimator.ofInt(0, header.getHeight()
				+ tempHeaderTopMargin);
		rbAnimator.setDuration(300);
		rbAnimator.setInterpolator(new AccelerateDecelerateInterpolator());
		rbAnimator
				.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
					@Override
					public void onAnimationUpdate(ValueAnimator animation) {
						int marginValue = Integer.parseInt(animation
								.getAnimatedValue().toString());
						/**
						 * 当刷新完毕后，进行headerView的隐藏
						 */
						setHeaderTopMarign(-marginValue + tempHeaderTopMargin);
					}
				});
		rbAnimator.addListener(new AnimatorListenerAdapter() {
			@Override
			public void onAnimationEnd(Animator animation) {
				if (currentStatus == STATUS_PULL_TO_REFRESH
						|| currentStatus == STATUS_REFRESH_FINISHED) {
					currentStatus = STATUS_REFRESH_FINISHED;
					return;
				}
				currentStatus = STATUS_REFRESH_FINISHED;
				header.start();
			}
		});
		rbAnimator.start();
	}



	/**
	 * 下拉刷新的监听器，使用下拉刷新的地方应该注册此监听器来获取刷新回调。
	 */
	public interface RefreshListener {
		/**
		 * 刷新时回调方法
		 */
		void onRefreshing();
	}

```

 这里面代码比较多，我们一步一步来看，根据上面提出的注意点2（滑动ListView时，是什么时候该下拉刷新，什么时候又是纯粹的往上滑动。也就是说只有在ListView滑动到顶部时，才是决定是否下拉。），也就是我们每次滑动时，都要去判断当前是否已经滑动到顶部，我们看checkAblePull方法，获取ListView中的第一个View，判断它是否显示在第一个位置并距离屏幕上方为0时，说明我们的ListView已经在顶部，这时获取手指滑动到的Y坐标；相反，如果我们的ListView并没有滑动到顶部，判断头部视图是否正确并进行重新设置位置。 

```
if (!ableToPull) {
			/**
			 * 只有ListView滚动到头的时候才允许下拉
			 */
			return false;
}
```

通过ableToPull进行下拉行为的判定。

接下来就是滑动到顶部时的一系列操作，查看滑动操作时的代码：

```
case MotionEvent.ACTION_MOVE:
			float currY = event.getRawY();
			float distance = currY - preDownY;
			float offsetY = distance * STICK_RATIO;
			if (distance <= 0
					&& headerLayoutParams.topMargin <= hideHeaderHeight) {
				return false;
			}
			if (distance < touchSlop) {
				return false;
			}

			header.start();
			if (headerLayoutParams.topMargin > 0) {
				currentStatus = STATUS_RELEASE_TO_REFRESH;
			} else {
				currentStatus = STATUS_PULL_TO_REFRESH;
			}

			// 通过偏移下拉头的topMargin值，来实现下拉效果
			setHeaderTopMarign((int) (offsetY + hideHeaderHeight));

			break;
```

这里主要分为顶部时的几种场景：

 1. 当手指向上滑动时（distance <= 0），并且头部视图小于等于头部视图本身的高度时，我们不做处理。
 2. 当滑动距离小于我们限定的最小距离，也不做处理。
 3. 最后判断头部视图是否完全显示出来。分两种状态，一种是已经将头部视图显示出来，并准松手释放；还有一种是处于下拉状态，但还没有完全显示出来。
 
 头部视图被逐渐往下来拉动的效果，查看setHeaderTopMargin方法：
 

```
	/**
	 * 给header设置topMargin参数
	 * 
	 * @param margin
	 */
	private void setHeaderTopMarign(int margin) {
		headerLayoutParams.topMargin = margin;
		header.setLayoutParams(headerLayoutParams);
	}
```

传入的值是我们在顶部，并向下滑动的Y坐标加上我们的头部视图的高度（hideHeaderHeight），这里面的高度一开始在onLayout进行初始化时被赋值给hideHeaderHeight，这里的（hideHeaderHeight）值一个负数，通过offsetY + hideHeaderHeight（【offsetY,负数】）不停地为topMargin 进行赋值。

最后就是手指离开屏幕时的操作：

```
case MotionEvent.ACTION_UP:
			if (currentStatus == STATUS_PULL_TO_REFRESH) {
				/**
				 * 下拉状态
				 */
				rollbackHeader();
			}
			if (currentStatus == STATUS_RELEASE_TO_REFRESH) {
				/**
				 * 头部向下拉时，松手后的逻辑
				 */
				scrollBackHeader();
			}
			break;
```

通过滑动过程中的状态值，在离开屏幕时进行相应的处理，当处于下拉状态时：

```
/**
	 * 回滚下拉刷新头部控件
	 */
	private void rollbackHeader() {
		tempHeaderTopMargin = headerLayoutParams.topMargin;
		ValueAnimator rbAnimator = ValueAnimator.ofInt(0, header.getHeight()
				+ tempHeaderTopMargin);
		rbAnimator.setDuration(300);
		rbAnimator.setInterpolator(new AccelerateDecelerateInterpolator());
		rbAnimator
				.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
					@Override
					public void onAnimationUpdate(ValueAnimator animation) {
						int marginValue = Integer.parseInt(animation
								.getAnimatedValue().toString());
						/**
						 * 当刷新完毕后，进行headerView的隐藏
						 */
						setHeaderTopMarign(-marginValue + tempHeaderTopMargin);
					}
				});
		rbAnimator.addListener(new AnimatorListenerAdapter() {
			@Override
			public void onAnimationEnd(Animator animation) {
				if (currentStatus == STATUS_PULL_TO_REFRESH
						|| currentStatus == STATUS_REFRESH_FINISHED) {
					currentStatus = STATUS_REFRESH_FINISHED;
					return;
				}
				currentStatus = STATUS_REFRESH_FINISHED;
				header.start();
			}
		});
		rbAnimator.start();
	}
```

当 头部处于向下拉时，松手后的逻辑：

```
	/**
	 * 回滚到头部刷新控件的高度，并触发后台刷新任务
	 */
	private void scrollBackHeader() {
		/**
		 * 当我们将头部布局一直往下来松手收，使它能平滑的回到顶部。
		 */
		ValueAnimator rbToHeaderAnimator = ValueAnimator.ofInt(
				headerLayoutParams.topMargin, 0);
		long duration = (long) (headerLayoutParams.topMargin * 1.1f) >= 0 ? (long) (headerLayoutParams.topMargin * 1.1f)
				: 0;
		rbToHeaderAnimator.setDuration(duration);
		rbToHeaderAnimator
				.setInterpolator(new AccelerateDecelerateInterpolator());
		rbToHeaderAnimator
				.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
					@Override
					public void onAnimationUpdate(ValueAnimator animation) {
						int marginValue = Integer.parseInt(animation
								.getAnimatedValue().toString());
						setHeaderTopMarign(marginValue);
					}
				});
		rbToHeaderAnimator.addListener(new AnimatorListenerAdapter() {
			@Override
			public void onAnimationEnd(Animator animation) {
				currentStatus = STATUS_REFRESHING;
				header.loading();
				Executors.newSingleThreadExecutor().submit(new Runnable() {
					@Override
					public void run() {
						if (mListener != null) {
							mListener.onRefreshing();
						}
					}
				});
			}
		});
		rbToHeaderAnimator.start();
	}
```

代码比较简单，都是对向下拉动头部视图时的操作，通过ValueAnimator时其向上滚动时，呈现平滑状态。

结束到这里，差不多已经结束了,以下给出PullToRefreshView的详细代码：

```
package com.example.pulltorefreshlistview.view;

import java.util.concurrent.Executors;

import android.animation.Animator;
import android.animation.AnimatorListenerAdapter;
import android.animation.ValueAnimator;
import android.content.Context;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;
import android.view.ViewConfiguration;
import android.view.animation.AccelerateDecelerateInterpolator;
import android.widget.LinearLayout;
import android.widget.ListView;

/**
 * Created by glh on 2016/3/2.
 */
public class PullToRefreshView extends LinearLayout implements
		View.OnTouchListener {

	/**
	 * 下拉状态
	 */
	public static final int STATUS_PULL_TO_REFRESH = 0;

	/**
	 * 释放准备刷新状态
	 */
	public static final int STATUS_RELEASE_TO_REFRESH = 1;

	/**
	 * 正在刷新状态
	 */
	public static final int STATUS_REFRESHING = 2;

	/**
	 * 刷新完成状态
	 */
	public static final int STATUS_REFRESH_FINISHED = 4;

	/**
	 * 下拉拖动的黏性比率
	 */
	private static final float STICK_RATIO = .65f;

	/**
	 * 下拉刷新的回调接口
	 */
	private RefreshListener mListener;

	/**
	 * 下拉头的View
	 */
	private PullHeaderView header;

	/**
	 * 需要去下拉刷新的ListView
	 */
	private ListView listView;

	/**
	 * 下拉控件布局参数
	 */
	private MarginLayoutParams headerLayoutParams;

	/**
	 * 下拉控件高度
	 */
	private int hideHeaderHeight;

	/**
	 * 当前状态
	 */
	private int currentStatus = STATUS_REFRESH_FINISHED;

	/**
	 * 手指按下时屏幕纵坐标
	 */
	private float preDownY;

	/**
	 * 在被判定为滚动之前用户手指可以移动的最大值
	 */
	private int touchSlop;

	/**
	 * 用于控制onLayout中的初始化只需加载一次
	 */
	private boolean once;

	/**
	 * 当前是否可以下拉，只有ListView滚动到头的时候才允许下拉
	 */
	private boolean ableToPull;

	private int tempHeaderTopMargin;

	public PullToRefreshView(Context context) {
		this(context, null);
	}

	public PullToRefreshView(Context context, AttributeSet attrs) {
		this(context, attrs, 0);
	}

	public PullToRefreshView(Context context, AttributeSet attrs,
			int defStyleAttr) {
		super(context, attrs, defStyleAttr);
		setOrientation(VERTICAL);
		touchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
		initView(context, attrs);
	}

	private void initView(Context context, AttributeSet attrs) {
		header = new PullHeaderView(context, attrs);
		header.start();
		addView(header, 0);
	}

	@Override
	protected void onLayout(boolean changed, int l, int t, int r, int b) {
		super.onLayout(changed, l, t, r, b);
		/**
		 * <pre>
		 * 只加载一次，用于将头部隐藏，这里面做了以下操作：
		 * 1、获取头部header的高度
		 * 2、获取头部header的布局参数MarginLayoutParams
		 * 3、为header设置布局参数（-header.getHeight()），进行隐藏。
		 * 4、拿到我们的ListView,并给它设置触摸事件。
		 * </pre>
		 */
		if (changed && !once) {
			hideHeaderHeight = -header.getHeight();
			headerLayoutParams = (MarginLayoutParams) header.getLayoutParams();
			headerLayoutParams.topMargin = hideHeaderHeight;
			listView = (ListView) getChildAt(1);
			listView.setOnTouchListener(this);
			once = true;
		}
	}

	@Override
	public boolean onTouch(View v, MotionEvent event) {
		checkAblePull(event);
		if (!ableToPull) {
			/**
			 * 只有ListView滚动到头的时候才允许下拉
			 */
			return false;
		}
		switch (event.getAction()) {
		case MotionEvent.ACTION_DOWN:
			preDownY = event.getRawY();
			break;
		case MotionEvent.ACTION_MOVE:
			float currY = event.getRawY();
			float distance = currY - preDownY;
			float offsetY = distance * STICK_RATIO;
			if (distance <= 0
					&& headerLayoutParams.topMargin <= hideHeaderHeight) {
				return false;
			}
			if (distance < touchSlop) {
				return false;
			}

			header.start();
			if (headerLayoutParams.topMargin > 0) {
				currentStatus = STATUS_RELEASE_TO_REFRESH;
			} else {
				currentStatus = STATUS_PULL_TO_REFRESH;
			}

			// 通过偏移下拉头的topMargin值，来实现下拉效果
			setHeaderTopMarign((int) (offsetY + hideHeaderHeight));

			break;
		case MotionEvent.ACTION_UP:
			if (currentStatus == STATUS_PULL_TO_REFRESH) {
				/**
				 * 下拉状态
				 */
				rollbackHeader();
			}
			if (currentStatus == STATUS_RELEASE_TO_REFRESH) {
				/**
				 * 头部向下拉时，松手后的逻辑
				 */
				scrollBackHeader();
			}
			break;
		}
		if (currentStatus == STATUS_PULL_TO_REFRESH
				|| currentStatus == STATUS_RELEASE_TO_REFRESH) {
			// 让ListView失去焦点, 不可被点击
			disableListView();
			return true;
		}
		return false;
	}

	//
	/**
	 * 给header设置topMargin参数
	 * 
	 * @param margin
	 */
	private void setHeaderTopMarign(int margin) {
		headerLayoutParams.topMargin = margin;
		header.setLayoutParams(headerLayoutParams);
	}

	/**
	 * 禁用ListView，让其失去焦点不可接受点击
	 */
	private void disableListView() {
		listView.setPressed(false);
		listView.setFocusable(false);
		listView.setFocusableInTouchMode(false);
	}

	/**
	 * 根据当前ListView的滚动状态来设定 {@link #ableToPull}
	 * 的值，每次都需要在onTouch中第一个执行，这样可以判断出当前应该是滚动ListView，还是应该进行下拉。
	 * 
	 * @param event
	 */
	private void checkAblePull(MotionEvent event) {
		View firstChild = listView.getChildAt(0);
		if (firstChild != null) {
			int firstVisiblePos = listView.getFirstVisiblePosition();
			if (firstVisiblePos == 0 && firstChild.getTop() == 0) {
				/**
				 * 如果首个元素的上边缘，距离父布局值为0，就说明ListView滚动到了最顶部，此时应该允许下拉刷新
				 */
				if (!ableToPull) {
					preDownY = event.getRawY();
				}
				ableToPull = true;
			} else { // 反之
				if (headerLayoutParams.topMargin != hideHeaderHeight) {
					setHeaderTopMarign(hideHeaderHeight);
				}
				ableToPull = false;
			}
		} else {
			/**
			 * 如果ListView中没有元素，也应该允许下拉刷新
			 */
			ableToPull = true;
		}
	}

	/**
	 * 给下拉刷新控件注册一个监听器。
	 * 
	 * @param listener
	 *            监听器的实现。
	 */
	public void setOnRefreshListener(RefreshListener listener) {
		mListener = listener;
	}

	/**
	 * 当所有的刷新逻辑完成后，记录调用一下，否则你的ListView将一直处于正在刷新状态。
	 */
	public void finishRefreshing() {
		header.end();
		/**
		 * 回滚下拉刷新头部控件
		 */
		rollbackHeader();
	}

	/**
	 * 回滚到头部刷新控件的高度，并触发后台刷新任务
	 */
	private void scrollBackHeader() {
		/**
		 * 当我们将头部布局一直往下来松手收，使它能平滑的回到顶部。
		 */
		ValueAnimator rbToHeaderAnimator = ValueAnimator.ofInt(
				headerLayoutParams.topMargin, 0);
		long duration = (long) (headerLayoutParams.topMargin * 1.1f) >= 0 ? (long) (headerLayoutParams.topMargin * 1.1f)
				: 0;
		rbToHeaderAnimator.setDuration(duration);
		rbToHeaderAnimator
				.setInterpolator(new AccelerateDecelerateInterpolator());
		rbToHeaderAnimator
				.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
					@Override
					public void onAnimationUpdate(ValueAnimator animation) {
						int marginValue = Integer.parseInt(animation
								.getAnimatedValue().toString());
						setHeaderTopMarign(marginValue);
					}
				});
		rbToHeaderAnimator.addListener(new AnimatorListenerAdapter() {
			@Override
			public void onAnimationEnd(Animator animation) {
				currentStatus = STATUS_REFRESHING;
				header.loading();
				Executors.newSingleThreadExecutor().submit(new Runnable() {
					@Override
					public void run() {
						if (mListener != null) {
							mListener.onRefreshing();
						}
					}
				});
			}
		});
		rbToHeaderAnimator.start();
	}

	/**
	 * 回滚下拉刷新头部控件
	 */
	private void rollbackHeader() {
		tempHeaderTopMargin = headerLayoutParams.topMargin;
		ValueAnimator rbAnimator = ValueAnimator.ofInt(0, header.getHeight()
				+ tempHeaderTopMargin);
		rbAnimator.setDuration(300);
		rbAnimator.setInterpolator(new AccelerateDecelerateInterpolator());
		rbAnimator
				.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
					@Override
					public void onAnimationUpdate(ValueAnimator animation) {
						int marginValue = Integer.parseInt(animation
								.getAnimatedValue().toString());
						/**
						 * 当刷新完毕后，进行headerView的隐藏
						 */
						setHeaderTopMarign(-marginValue + tempHeaderTopMargin);
					}
				});
		rbAnimator.addListener(new AnimatorListenerAdapter() {
			@Override
			public void onAnimationEnd(Animator animation) {
				if (currentStatus == STATUS_PULL_TO_REFRESH
						|| currentStatus == STATUS_REFRESH_FINISHED) {
					currentStatus = STATUS_REFRESH_FINISHED;
					return;
				}
				currentStatus = STATUS_REFRESH_FINISHED;
				header.start();
			}
		});
		rbAnimator.start();
	}

	/**
	 * 下拉刷新的监听器，使用下拉刷新的地方应该注册此监听器来获取刷新回调。
	 */
	public interface RefreshListener {
		/**
		 * 刷新时回调方法
		 */
		void onRefreshing();
	}

}
```

```
package com.example.pulltorefreshlistview;

import java.util.ArrayList;
import java.util.List;

import com.example.pulltorefreshlistview.view.PullToRefreshView;

import android.app.Activity;
import android.os.Bundle;
import android.os.Handler;
import android.os.Message;
import android.widget.ArrayAdapter;
import android.widget.ListView;
import android.widget.Toast;

public class MainActivity extends Activity {

	private PullToRefreshView refreshView;

	private ListView listView;

	private List<String> dataList;

	private ArrayAdapter<String> arrayAdapter;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		refreshView = (PullToRefreshView) findViewById(R.id.refresh_hit_block);

		listView = (ListView) findViewById(R.id.list_view);

		arrayAdapter = new ArrayAdapter<String>(this,
				android.R.layout.simple_expandable_list_item_1, createDate());

		listView.setAdapter(arrayAdapter);
		refreshView
				.setOnRefreshListener(new PullToRefreshView.RefreshListener() {
					@Override
					public void onRefreshing() {
						try {
							// 模拟网络请求耗时动作
							Thread.sleep(2000);
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
						mHandler.sendEmptyMessage(0);
					}
				});
	}

	private Handler mHandler = new Handler() {
		@Override
		public void handleMessage(Message msg) {
			dataList.add("新增内容");
			arrayAdapter.notifyDataSetChanged();
			refreshView.finishRefreshing();
			Toast.makeText(MainActivity.this, "刷新成功!", Toast.LENGTH_SHORT)
					.show();
		}
	};

	private List<String> createDate() {
		dataList = new ArrayList<String>();
		dataList.add("Item1");
		dataList.add("Item2");
		dataList.add("Item3");
		dataList.add("Item4");
		dataList.add("Item5");
		return dataList;
	}
}

```

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <com.example.pulltorefreshlistview.view.PullToRefreshView
        android:id="@+id/refresh_hit_block"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" >

        <ListView
            android:id="@+id/list_view"
            android:layout_width="fill_parent"
            android:layout_height="fill_parent"
            android:scrollbars="none" >
        </ListView>
    </com.example.pulltorefreshlistview.view.PullToRefreshView>

</RelativeLayout>
```
