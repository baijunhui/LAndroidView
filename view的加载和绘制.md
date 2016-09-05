学习android界面显示

界面显示分为两步，一是添加到window上，二是view的加载绘制

一、xml布局文件添加到activity上及window上

我们都知道在Activity的oncreate()里通过setContentView(id or view)可以设置手机图片的显示，具体怎么显示的呢，我们这里通过源码来探索下

1、通过avtivity里的setContentView方法
	/**
     * Set the activity content from a layout resource.  The resource will be
     * inflated, adding all top-level views to the activity.
     *
     * @param layoutResID Resource ID to be inflated.
     *
     * @see #setContentView(android.view.View)
     * @see #setContentView(android.view.View, android.view.ViewGroup.LayoutParams)
     */
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

    通常我们都是用layoutId设置ui，我们也可以通过直接传入view来设置ui，不同只是传入参数不同而已
    下面我们通过setContentView(layoutId)来分析

 2、我们来分析第一行getWindow().setContentView(id), getWindow()方法拿到是activity里的一个变量mWindow

 	/**
     * Retrieve the current {@link android.view.Window} for the activity.
     * This can be used to directly access parts of the Window API that
     * are not available through Activity/Screen.
     *
     * @return Window The current window, or null if the activity is not
     *         visual.
     */
    public Window getWindow() {
        return mWindow;
    }

	通过getWindow方法获得activity里的变量mWindow,下面看看mWindow是在何时赋值的。

 	final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        //api 23
        mWindow = new PhoneWindow(this);
        //api 21
        mWindow = PolicyManager.makeNewWindow(this);

        。。。
    }

    attach方法是在ActivityThread里生命周期执行oncreate之前, 获取到activity，调用activity.attach()

    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    	...
    	if (activity != null) {
            。。。
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.voiceInteractor);

            。。。
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            。。。
        }

    }

    下面我们来看看phoneWindow的setContentView方法

    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        。。。
    }
    主要有两个关键一个是installDecor(),另一个是mLayoutInflater.inflate(id, mContentParent);
    下面我们先来看第一个

3、installDecor()方法,里面代码很多，我们看关键代码

	private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            ...
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

           	...
        }
    }
    主要就是两个，一个是生成mDecor，另一个是生成mContentParent，他们到底是什么呢，看注释

    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;

    // This is the view in which the window contents are placed. It is either
    // mDecor itself, or a child of mDecor where the contents go.
    private ViewGroup mContentParent;

4、下面我们来看看怎么生成DecorView顶级view
	protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }
    直接new的生成的，DecorView是PhoneWindow的内部类，继承FrameLayout，这样setContentView里就有了一个FrameLayout

5、接着看怎么生成mContentParent, 传入了刚生成的DecorView

	protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.

        TypedArray a = getWindowStyle();

        ...
        //拿到activity窗口的一些属性

        if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
            requestFeature(FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestFeature(FEATURE_ACTION_BAR);
        }

        。。。
        //设置状态栏颜色
        if (!mForcedStatusBarColor) {
            mStatusBarColor = a.getColor(R.styleable.Window_statusBarColor, 0xFF000000);
        }
        //设置导航栏颜色
        if (!mForcedNavigationBarColor) {
            mNavigationBarColor = a.getColor(R.styleable.Window_navigationBarColor, 0xFF000000);
        }

        
        。。。

        //设置动画
        if (params.windowAnimations == 0) {
            params.windowAnimations = a.getResourceId(
                    R.styleable.Window_windowAnimationStyle, 0);
        }

        。。。

        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();
        // System.out.println("Features: 0x" + Integer.toHexString(features));
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_title_icons;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println("Title Icons!");
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            // Special case for a window with only a progress bar (and title).
            // XXX Need to have a no-title version of embedded windows.
            layoutResource = R.layout.screen_progress;
            // System.out.println("Progress!");
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_custom_title;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                layoutResource = a.getResourceId(
                        R.styleable.Window_windowActionBarFullscreenDecorLayout,
                        R.layout.screen_action_bar);
            } else {
                layoutResource = R.layout.screen_title;
            }
            // System.out.println("Title!");
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            // Embedded, so no decoration is needed.
            layoutResource = R.layout.screen_simple;
            // System.out.println("Simple!");
        }

        mDecor.startChanging();

        //通过LayoutInflater把resource生成布局
        View in = mLayoutInflater.inflate(layoutResource, null);
        //把xml文件生成的布局添加到decorView上
        //至此PhoneWindow有了DecorView，DecorView里有了布局
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        。。。

        //设置背景，title到activity
        // Remaining setup -- of background and title -- that only applies
        // to top-level windows.
        。。。

        return contentParent;
    }

6、下面我们来看下，刚才的layoutResource是什么，拿R.layout.screen_simple这个布局来看下

		<?xml version="1.0" encoding="utf-8"?>
	<!--
	/* //device/apps/common/assets/res/layout/screen_simple.xml
	**
	** Copyright 2006, The Android Open Source Project
	**
	** Licensed under the Apache License, Version 2.0 (the "License"); 
	** you may not use this file except in compliance with the License. 
	** You may obtain a copy of the License at 
	**
	**     http://www.apache.org/licenses/LICENSE-2.0 
	**
	** Unless required by applicable law or agreed to in writing, software 
	** distributed under the License is distributed on an "AS IS" BASIS, 
	** WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. 
	** See the License for the specific language governing permissions and 
	** limitations under the License.
	*/

	This is an optimized layout for a screen, with the minimum set of features
	enabled.
	-->

	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:fitsSystemWindows="true"
	    android:orientation="vertical">
	    <ViewStub android:id="@+id/action_mode_bar_stub"
	              android:inflatedId="@+id/action_mode_bar"
	              android:layout="@layout/action_mode_bar"
	              android:layout_width="match_parent"
	              android:layout_height="wrap_content"
	              android:theme="?attr/actionBarTheme" />
	    <FrameLayout
	         android:id="@android:id/content"
	         android:layout_width="match_parent"
	         android:layout_height="match_parent"
	         android:foregroundInsidePadding="false"
	         android:foregroundGravity="fill_horizontal|top"
	         android:foreground="?android:attr/windowContentOverlay" />
	</LinearLayout>

	DecorView里添加了线性布局LinearLayout，线性布局里有两个组件，ViewSub是懒加载，默认是不加载的，FrameLayout是id为content的控件，再倒着看上面几步：
	ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
	mContentParent = generateLayout(mDecor);
	最终mContentParent就是拿到的这个FrameLayout

	再往前几步 mLayoutInflater.inflate(layoutResID, mContentParent); LayoutInflater加载器，把传进来的layoutResId加载到mContentParent里，至此xml布局文件就加载到了activity里了


下面来总结下流程：
activity setContentView -> window setContentView -> PhoneWindow setContentView -> PhoneWindow installDecor -> 
PhoneWinodw generateDecor -> PhoneWindow generateLayout -> PhoneWindow inflate.inflate


Activity里有个抽象的Window,它的实现类是PhoneWindow,PhoneWindow里有个继承FlameLayout类的DecorView类，在DecorView中添加根布局，根布局里有个id是content的FrameLayout，我们在activity里的setContentView添加的xml布局直接添加到了id是content的FrameLayout里了。






二、解析布局文件
	我们都知道通过LayoutInflater.inflate()可以解析布局文件，具体怎么解析的呢，我们下面来分析下：
	上面我们有用mLayoutInflater.inflate(layoutResID, mContentParent); 加载了主布局文件，把传入的layout布局加载到了content上

1、从mLayoutInflater.inflate(layoutResID, mContentParent)追踪解析过程，在LayoutInflater里第一个是这个
	/**
     * Inflate a new view hierarchy from the specified xml resource. Throws
     * {@link InflateException} if there is an error.
     * 
     * @param resource ID for an XML layout resource to load (e.g.,
     *        <code>R.layout.main_page</code>)
     * @param root Optional view to be the parent of the generated hierarchy.
     * @return The root View of the inflated hierarchy. If root was supplied,
     *         this is the root View; otherwise it is the root of the inflated
     *         XML file.
     */
    public View inflate(int resource, ViewGroup root) {
        return inflate(resource, root, root != null);
    }

2、 inflate()方法，我们经常使用的一种解析xml的方法
	/**
     * Inflate a new view hierarchy from the specified xml resource. Throws
     * {@link InflateException} if there is an error.
     * 
     * @param resource ID for an XML layout resource to load (e.g.,
     *        <code>R.layout.main_page</code>)
     * @param root Optional view to be the parent of the generated hierarchy (if
     *        <em>attachToRoot</em> is true), or else simply an object that
     *        provides a set of LayoutParams values for root of the returned
     *        hierarchy (if <em>attachToRoot</em> is false.)
     * @param attachToRoot Whether the inflated hierarchy should be attached to
     *        the root parameter? If false, root is only used to create the
     *        correct subclass of LayoutParams for the root view in the XML.
     * @return The root View of the inflated hierarchy. If root was supplied and
     *         attachToRoot is true, this is root; otherwise it is the root of
     *         the inflated XML file.
     */

    public View inflate(int resource, ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
    这个是我们经常使用的解析的一种方式，传入id，root，boolean值。通过Resources拿到XmlResourceParser解析工具

3、下面来看看inflate()方法解析，找到root节点，并解析出root view
	
	public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context)mConstructorArgs[0];
            mConstructorArgs[0] = mContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }

                //通过不同节点判断处理流程
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, attrs, false, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, attrs, false);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }
                    // Inflate all children under temp
                    rInflate(parser, temp, attrs, true, true);
                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (IOException e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                        + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }

            Trace.traceEnd(Trace.TRACE_TAG_VIEW);

            return result;
        }
    }

    上面这段代码主要作用是根据根节点创建一个root view，while寻欢是查找根节点，如果是merge，则直接调用rInflate()遍历所有的节点，
    如果不是merge，则用createViewFromTag()根据名字创建一个root view，然后调用rInflate()方法遍历所有节点，创建子view添加到root上。

    主要有createViewFromTag()和 rInflate()这两个方法，我们下面一一分析

4、createViewFromTag()这个方法主要是根据名称节点来创建View实例，通过反射来创建实例对象的

	View createViewFromTag(View parent, String name, AttributeSet attrs, boolean inheritContext) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        Context viewContext;
        if (parent != null && inheritContext) {
            viewContext = parent.getContext();
        } else {
            viewContext = mContext;
        }

        // Apply a theme wrapper, if requested.
        final TypedArray ta = viewContext.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            viewContext = new ContextThemeWrapper(viewContext, themeResId);
        }
        ta.recycle();

        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(viewContext, attrs);
        }

        if (DEBUG) System.out.println("******** Creating view: " + name);

        try {
            View view;
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, viewContext, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, viewContext, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, viewContext, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = viewContext;
                try {
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            if (DEBUG) System.out.println("Created view is: " + view);
            return view;

        } catch (InflateException e) {
            throw e;

        } catch (ClassNotFoundException e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;

        } catch (Exception e) {
            InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;
        }
    }


    //通过反射来创建view实例
    /**
     * Low-level function for instantiating a view by name. This attempts to
     * instantiate a view class of the given <var>name</var> found in this
     * LayoutInflater's ClassLoader.
     * 
     * <p>
     * There are two things that can happen in an error case: either the
     * exception describing the error will be thrown, or a null will be
     * returned. You must deal with both possibilities -- the former will happen
     * the first time createView() is called for a class of a particular name,
     * the latter every time there-after for that class name.
     * 
     * @param name The full name of the class to be instantiated.
     * @param attrs The XML attributes supplied for this instance.
     * 
     * @return View The newly instantiated view, or null.
     */
    public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        Class<? extends View> clazz = null;

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                
                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                constructor = clazz.getConstructor(mConstructorSignature);
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) : name).asSubclass(View.class);
                        
                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
            }

            Object[] args = mConstructorArgs;
            args[1] = attrs;

            constructor.setAccessible(true);
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            return view;

        ，，，
    }

5、rInflate()这个方法是循环遍历所有节点，并把创建的view添加到root上去

	void rInflate(XmlPullParser parser, View parent, final AttributeSet attrs,
            boolean finishInflate, boolean inheritContext) throws XmlPullParserException,
            IOException {

        final int depth = parser.getDepth();
        int type;

        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, parent, attrs, inheritContext);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else {

            	//递归向下查询生成view
                final View view = createViewFromTag(parent, name, attrs, inheritContext);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflate(parser, view, attrs, true, true);
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) parent.onFinishInflate();
    }

	viewGroup.add(view, params)把解析出来的所有的view都添加到了viewGroup上了，最终加到了DecorView中的id是content的FrameLayout上的。


总结：
	public View inflate(int resource, ViewGroup root){}

	public View inflate(int resource, ViewGroup root, boolean attachToRoot) {}

	这是LayoutInflate对外提供的两个接口 
	1)当root=null时，attachToRoot失效，root不起作用
	2)当root!=null时，attachToRoot=false,xml解析出的view不添加到root上，root不起作用
	3)当root!=null时,attachToRoot=true,xml会添加到根目录root上



三、分析view的绘制

1、	这个就要说到activity生命周期过程中onresume的ActivityThread handleResumeActivity

	final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        
        ...

        //回调到activity的onresume方法的
        // TODO Push resumeArgs into the activity for consideration
        ActivityClientRecord r = performResumeActivity(token, clearHide);

        if (r != null) {
            。。。
            //这是处理window界面显示的
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }

           	。。。

            // The window is now visible if it has been added, we are not
            // simply finishing, and we are not starting another activity.
            if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                。。。
            }

            。。。
    }

    从上面代码我们可以看到View decor = r.window.getDecorView();拿到我们添加到window上的DecorView设置为InVisible，并且把DecorView赋值到activity的DecorView上。然后拿到iewManager wm = a.getWindowManager();windowMannager，把Decorview添加到了wm上了。WindowManager是一个接口，它的实现类是WindowManagerImpl，所以我们接下来看下WindowManagerImpl里怎么实现addView方法的

2、WindowManagerImpl.addView()方法

	@Override
    public void addView(View view, ViewGroup.LayoutParams params) {
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }

    可以看到这个里面只有一个调用了WindowManagerGlobal的addview方法，多传了两个参数，这两个参数是在构造函数的时候传进去的，下面看下global的addview方法

3、WindowManagerGlobal.addView()方法

	public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {

        //判断传入参数是否合法
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent and we're running on L or above (or in the
            // system context), assume we want hardware acceleration.
            final Context context = view.getContext();
            if (context != null
                    && context.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }

            //直接new了个ViewRoomImpl
            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }

    我们可以看到，里面直接new了个ViewRoomImpl对象root，并且把view添加到root上了，并且给view设置上了params
    这里有三个缓存，mViews,mRoots, mParams这三个

4、ViewRoomImpl.setView()我们来看下这个方法
	
	/**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                ....

                mAdded = true;
                int res; /* = WindowManagerImpl.ADD_OKAY; */

                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                requestLayout();
                
                ...

            }
        }
    }

5、requestLayout()

	@Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

6、scheduleTraversals()

	void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
        }
    }


    inal class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();


7、doTraversal（）

	void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "performTraversals");
            try {
                performTraversals();
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }

8、performTraversals()

	private void performTraversals() {
        // cache mView since it is used so much below...
        final View host = mView;

        ...

        if (host == null || !mAdded)
            return;

        //立马遍历
        mIsInTraversal = true;
        //马上绘制
        mWillDrawSoon = true;
        boolean windowSizeMayChange = false;
        boolean newSurface = false;
        boolean surfaceChanged = false;
        WindowManager.LayoutParams lp = mWindowAttributes;

        //decorview的宽高
        int desiredWindowWidth;
        int desiredWindowHeight;

        。。。

        Rect frame = mWinFrame;
        //是不是首次，在初始化的时候已经赋值
        // true for the first time the view is added
        if (mFirst) {
            mFullRedrawNeeded = true;
            mLayoutRequested = true;

           	//窗口是否有状态栏，有的话，decorview宽高就是减去状态栏的
            if (lp.type == WindowManager.LayoutParams.TYPE_STATUS_BAR_PANEL
                    || lp.type == WindowManager.LayoutParams.TYPE_INPUT_METHOD) {
                // NOTE -- system code, won't try to do compat mode.
                Point size = new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth = size.x;
                desiredWindowHeight = size.y;
            } else {
                DisplayMetrics packageMetrics =
                    mView.getContext().getResources().getDisplayMetrics();
                desiredWindowWidth = packageMetrics.widthPixels;
                desiredWindowHeight = packageMetrics.heightPixels;
            }

            。。。

        } else {

            ...

        }

        。。。

        if (mFist || ...) {

        	...

            if (!mStopped) {
                boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                        (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
                if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged) {

                    //view的宽高，mWidth,mHeight是窗口的宽高，lp.width,lp.height是DecorView是宽高
                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

                    //测量view，执行测量操作
                     // Ask host how big it wants to be
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                    。。。

                    layoutRequested = true;
                }
            }
        } else {
            。。。
        }

        final boolean didLayout = layoutRequested && !mStopped;
        boolean triggerGlobalLayoutListener = didLayout
                || mAttachInfo.mRecomputeGlobalAttributes;
        if (didLayout) {

        	//执行布局操作
            performLayout(lp, desiredWindowWidth, desiredWindowHeight);

            。。。

        }

        。。。

        boolean skipDraw = false;

        。。。

        mFirst = false;
        mWillDrawSoon = false;
        mNewSurfaceNeeded = false;
        mViewVisibility = viewVisibility;

        。。。

        boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() ||
                viewVisibility != View.VISIBLE;

        if (!cancelDraw && !newSurface) {
            if (!skipDraw || mReportNextDraw) {

                。。。

                //执行绘制操作
                performDraw();
            }
        } else {
            if (viewVisibility == View.VISIBLE) {
                // Try again
                scheduleTraversals();
            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                。。。
            }
        }

        mIsInTraversal = false;
    }

   	中间经过一些代码，终于到了重量级的代码了：
   	performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);执行测量操作
   	performLayout(lp, desiredWindowWidth, desiredWindowHeight);执行布局操作
   	performDraw();执行绘制操作
   	下面我们来分析下怎么执行的，分析之前，我们先看下获取到root view的测量模式的

   	private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }

9、performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);测量

	1）private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
    该代码执行了view的measure方法，view的测量方法

   2）下面是view的measure方法，可以看出，measure是final的，子类不可以继承

   public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

        。。。

        //如果不一致则重新测量
        if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {

            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                    mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

        //重新赋值，以便下次进行测量判断
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }

   3） view的onMeasure()方法

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }

    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }


    4）FrameLayout的onMeasure方法,里面没有measure方法

    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        ...

        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        。。。
    }

    5）ViewGroup的measureChildWithMargins方法，里面调用了view的measure方法，测量每个view的大小

	    protected void measureChildWithMargins(View child,
	            int parentWidthMeasureSpec, int widthUsed,
	            int parentHeightMeasureSpec, int heightUsed) {
	        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

	        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
	                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
	                        + widthUsed, lp.width);
	        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
	                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
	                        + heightUsed, lp.height);

	        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
	    }

   	6）viewGroup下每个子view的测试规格

	    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
	        int specMode = MeasureSpec.getMode(spec);
	        int specSize = MeasureSpec.getSize(spec);

	        int size = Math.max(0, specSize - padding);

	        int resultSize = 0;
	        int resultMode = 0;

	        switch (specMode) {
	        // Parent has imposed an exact size on us
	        case MeasureSpec.EXACTLY:
	            if (childDimension >= 0) {
	                resultSize = childDimension;
	                resultMode = MeasureSpec.EXACTLY;
	            } else if (childDimension == LayoutParams.MATCH_PARENT) {
	                // Child wants to be our size. So be it.
	                resultSize = size;
	                resultMode = MeasureSpec.EXACTLY;
	            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
	                // Child wants to determine its own size. It can't be
	                // bigger than us.
	                resultSize = size;
	                resultMode = MeasureSpec.AT_MOST;
	            }
	            break;

	        // Parent has imposed a maximum size on us
	        case MeasureSpec.AT_MOST:
	            if (childDimension >= 0) {
	                // Child wants a specific size... so be it
	                resultSize = childDimension;
	                resultMode = MeasureSpec.EXACTLY;
	            } else if (childDimension == LayoutParams.MATCH_PARENT) {
	                // Child wants to be our size, but our size is not fixed.
	                // Constrain child to not be bigger than us.
	                resultSize = size;
	                resultMode = MeasureSpec.AT_MOST;
	            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
	                // Child wants to determine its own size. It can't be
	                // bigger than us.
	                resultSize = size;
	                resultMode = MeasureSpec.AT_MOST;
	            }
	            break;

	        // Parent asked to see how big we want to be
	        case MeasureSpec.UNSPECIFIED:
	            if (childDimension >= 0) {
	                // Child wants a specific size... let him have it
	                resultSize = childDimension;
	                resultMode = MeasureSpec.EXACTLY;
	            } else if (childDimension == LayoutParams.MATCH_PARENT) {
	                // Child wants to be our size... find out how big it should
	                // be
	                resultSize = 0;
	                resultMode = MeasureSpec.UNSPECIFIED;
	            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
	                // Child wants to determine its own size.... find out how
	                // big it should be
	                resultSize = 0;
	                resultMode = MeasureSpec.UNSPECIFIED;
	            }
	            break;
	        }
	        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
	    }


10、performLayout(lp, desiredWindowWidth, desiredWindowHeight);

	1）performLayout()直接调用这个方法
		private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
		        int desiredWindowHeight) {
		    ...

		    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
		    try {
		        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

		        ...
		       
		    } finally {
		        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
		    }
		    mInLayout = false;
		}

    2）view的layout()方法

	    @SuppressWarnings({"unchecked"})
	    public void layout(int l, int t, int r, int b) {
	        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
	            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
	            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
	        }

	        int oldL = mLeft;
	        int oldT = mTop;
	        int oldB = mBottom;
	        int oldR = mRight;

	        boolean changed = isLayoutModeOptical(mParent) ?
	                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

	        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
	            onLayout(changed, l, t, r, b);
	            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

	            ListenerInfo li = mListenerInfo;
	            if (li != null && li.mOnLayoutChangeListeners != null) {
	                ArrayList<OnLayoutChangeListener> listenersCopy =
	                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
	                int numListeners = listenersCopy.size();
	                for (int i = 0; i < numListeners; ++i) {
	                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
	                }
	            }
	        }

	        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
	        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
	    }

    3) setFrame方法

	    protected boolean setFrame(int left, int top, int right, int bottom) {
	        boolean changed = false;

	        if (DBG) {
	            Log.d("View", this + " View.setFrame(" + left + "," + top + ","
	                    + right + "," + bottom + ")");
	        }

	        //判断和之前位置是否相同
	        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
	            changed = true;

	            // Remember our drawn bit
	            int drawn = mPrivateFlags & PFLAG_DRAWN;

	            int oldWidth = mRight - mLeft;
	            int oldHeight = mBottom - mTop;
	            int newWidth = right - left;
	            int newHeight = bottom - top;
	            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

	            // Invalidate our old position
	            invalidate(sizeChanged);

	            //给全局变量位置信息赋值了
	            mLeft = left;
	            mTop = top;
	            mRight = right;
	            mBottom = bottom;
	            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);

	            mPrivateFlags |= PFLAG_HAS_BOUNDS;


	            if (sizeChanged) {
	                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
	            }

	            。。。

	        }
	        return changed;
	    }

	    这步之后，我们才可以通过view.getWidth()拿到view的宽，到这步布局已经完成，为什么还有下面的onLayout()方法呢，下面看下这个方法

    4）view.onLayout()方法

	    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
	    }

	    空的方法，没有任何内容，意料之中，view中的布局由父类视图布局决定，也就是ViewGroup决定的，所有一般自定义view，不需要重写onLayout，但是自定义ViewGroup一定要重写onLayout方法，它是抽象的

    5）FrameLayout.onLayout()方法

	    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
	        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
	    }

	    void layoutChildren(int left, int top, int right, int bottom,
	                                  boolean forceLeftGravity) {
	        final int count = getChildCount();

	        final int parentLeft = getPaddingLeftWithForeground();
	        final int parentRight = right - left - getPaddingRightWithForeground();

	        final int parentTop = getPaddingTopWithForeground();
	        final int parentBottom = bottom - top - getPaddingBottomWithForeground();

	        mForegroundBoundsChanged = true;
	        
	        //遍历所有的子view
	        for (int i = 0; i < count; i++) {
	            final View child = getChildAt(i);
	            //当view时GONE时，不进行布局，不占用空间
	            if (child.getVisibility() != GONE) {
	                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

	                final int width = child.getMeasuredWidth();
	                final int height = child.getMeasuredHeight();

	                int childLeft;
	                int childTop;

	                int gravity = lp.gravity;
	                if (gravity == -1) {
	                    gravity = DEFAULT_CHILD_GRAVITY;
	                }

	                final int layoutDirection = getLayoutDirection();
	                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
	                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

	                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
	                    case Gravity.CENTER_HORIZONTAL:
	                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
	                        lp.leftMargin - lp.rightMargin;
	                        break;
	                    case Gravity.RIGHT:
	                        if (!forceLeftGravity) {
	                            childLeft = parentRight - width - lp.rightMargin;
	                            break;
	                        }
	                    case Gravity.LEFT:
	                    default:
	                        childLeft = parentLeft + lp.leftMargin;
	                }

	                switch (verticalGravity) {
	                    case Gravity.TOP:
	                        childTop = parentTop + lp.topMargin;
	                        break;
	                    case Gravity.CENTER_VERTICAL:
	                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
	                        lp.topMargin - lp.bottomMargin;
	                        break;
	                    case Gravity.BOTTOM:
	                        childTop = parentBottom - height - lp.bottomMargin;
	                        break;
	                    default:
	                        childTop = parentTop + lp.topMargin;
	                }

	                //进行子布局
	                child.layout(childLeft, childTop, childLeft + width, childTop + height);
	            }
	        }
	    }

	    我们可以看出来，FrameLayout的onLayout方法里只是循环调用了子view的layout方法

    6)ViewGroup.onLayout()是个抽象的方法

	    protected abstract void onLayout(boolean changed,
	            int l, int t, int r, int b);


	    所有继承ViewGroup的子类都必须实现onLayout方法

11、performDraw()方法

	1）private void performDraw() {
        。。。
        try {
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        。。。
    }

    2）draw() -> drawSoftware()

    	private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {

	        // Draw with software renderer.
	        final Canvas canvas;
	        try {
	            final int left = dirty.left;
	            final int top = dirty.top;
	            final int right = dirty.right;
	            final int bottom = dirty.bottom;

	            //从Surface中获取到canvas
	            canvas = mSurface.lockCanvas(dirty);

	            。。。
	        }

	        try {
	            。。。
	            try {
	            	//平移画布
	                canvas.translate(-xoff, -yoff);
	                if (mTranslator != null) {
	                    mTranslator.translateCanvas(canvas);
	                }
	                canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
	                attachInfo.mSetIgnoreDirtyState = false;

	                //调用view的draw方法绘制图形
	                mView.draw(canvas);
	            } finally {
	                if (!attachInfo.mSetIgnoreDirtyState) {
	                    // Only clear the flag if it was not set during the mView.draw() call
	                    attachInfo.mIgnoreDirtyState = false;
	                }
	            }
	        } finally {
	            ...
	        }
	        return true;
	    }

    从Surface中获取到画布，最后调用view的draw方法绘制，可见绘制的图形view在Surface上了。

    3）view。draw（）方法

	    public void draw(Canvas canvas) {
	        final int privateFlags = mPrivateFlags;
	        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
	                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
	        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

	        /*
	         * Draw traversal performs several drawing steps which must be executed
	         * in the appropriate order:
	         *
	         *      1. Draw the background
	         *      2. If necessary, save the canvas' layers to prepare for fading
	         *      3. Draw view's content
	         *      4. Draw children
	         *      5. If necessary, draw the fading edges and restore layers
	         *      6. Draw decorations (scrollbars for instance)
	         */

	        // Step 1, draw the background, if needed
	        int saveCount;

	        if (!dirtyOpaque) {
	            drawBackground(canvas);
	        }

	        // skip step 2 & 5 if possible (common case)
	        final int viewFlags = mViewFlags;
	        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
	        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
	        if (!verticalEdges && !horizontalEdges) {
	            // Step 3, draw the content
	            if (!dirtyOpaque) onDraw(canvas);

	            // Step 4, draw the children
	            dispatchDraw(canvas);

	            // Step 6, draw decorations (scrollbars)
	            onDrawScrollBars(canvas);

	            if (mOverlay != null && !mOverlay.isEmpty()) {
	                mOverlay.getOverlayView().dispatchDraw(canvas);
	            }

	            // we're done...
	            return;
	        }

	        /*
	         * Here we do the full fledged routine...
	         * (this is an uncommon case where speed matters less,
	         * this is why we repeat some of the tests that have been
	         * done above)
	         */

	        boolean drawTop = false;
	        boolean drawBottom = false;
	        boolean drawLeft = false;
	        boolean drawRight = false;

	        float topFadeStrength = 0.0f;
	        float bottomFadeStrength = 0.0f;
	        float leftFadeStrength = 0.0f;
	        float rightFadeStrength = 0.0f;

	        // Step 2, save the canvas' layers
	        int paddingLeft = mPaddingLeft;

	        final boolean offsetRequired = isPaddingOffsetRequired();
	        if (offsetRequired) {
	            paddingLeft += getLeftPaddingOffset();
	        }

	        int left = mScrollX + paddingLeft;
	        int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
	        int top = mScrollY + getFadeTop(offsetRequired);
	        int bottom = top + getFadeHeight(offsetRequired);

	        if (offsetRequired) {
	            right += getRightPaddingOffset();
	            bottom += getBottomPaddingOffset();
	        }

	        final ScrollabilityCache scrollabilityCache = mScrollCache;
	        final float fadeHeight = scrollabilityCache.fadingEdgeLength;
	        int length = (int) fadeHeight;

	        // clip the fade length if top and bottom fades overlap
	        // overlapping fades produce odd-looking artifacts
	        if (verticalEdges && (top + length > bottom - length)) {
	            length = (bottom - top) / 2;
	        }

	        // also clip horizontal fades if necessary
	        if (horizontalEdges && (left + length > right - length)) {
	            length = (right - left) / 2;
	        }

	        if (verticalEdges) {
	            topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
	            drawTop = topFadeStrength * fadeHeight > 1.0f;
	            bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
	            drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
	        }

	        if (horizontalEdges) {
	            leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
	            drawLeft = leftFadeStrength * fadeHeight > 1.0f;
	            rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
	            drawRight = rightFadeStrength * fadeHeight > 1.0f;
	        }

	        saveCount = canvas.getSaveCount();

	        int solidColor = getSolidColor();
	        if (solidColor == 0) {
	            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

	            if (drawTop) {
	                canvas.saveLayer(left, top, right, top + length, null, flags);
	            }

	            if (drawBottom) {
	                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
	            }

	            if (drawLeft) {
	                canvas.saveLayer(left, top, left + length, bottom, null, flags);
	            }

	            if (drawRight) {
	                canvas.saveLayer(right - length, top, right, bottom, null, flags);
	            }
	        } else {
	            scrollabilityCache.setFadeColor(solidColor);
	        }

	        // Step 3, draw the content
	        if (!dirtyOpaque) onDraw(canvas);

	        // Step 4, draw the children
	        dispatchDraw(canvas);

	        // Step 5, draw the fade effect and restore layers
	        final Paint p = scrollabilityCache.paint;
	        final Matrix matrix = scrollabilityCache.matrix;
	        final Shader fade = scrollabilityCache.shader;

	        if (drawTop) {
	            matrix.setScale(1, fadeHeight * topFadeStrength);
	            matrix.postTranslate(left, top);
	            fade.setLocalMatrix(matrix);
	            p.setShader(fade);
	            canvas.drawRect(left, top, right, top + length, p);
	        }

	        if (drawBottom) {
	            matrix.setScale(1, fadeHeight * bottomFadeStrength);
	            matrix.postRotate(180);
	            matrix.postTranslate(left, bottom);
	            fade.setLocalMatrix(matrix);
	            p.setShader(fade);
	            canvas.drawRect(left, bottom - length, right, bottom, p);
	        }

	        if (drawLeft) {
	            matrix.setScale(1, fadeHeight * leftFadeStrength);
	            matrix.postRotate(-90);
	            matrix.postTranslate(left, top);
	            fade.setLocalMatrix(matrix);
	            p.setShader(fade);
	            canvas.drawRect(left, top, left + length, bottom, p);
	        }

	        if (drawRight) {
	            matrix.setScale(1, fadeHeight * rightFadeStrength);
	            matrix.postRotate(90);
	            matrix.postTranslate(right, top);
	            fade.setLocalMatrix(matrix);
	            p.setShader(fade);
	            canvas.drawRect(right - length, top, right, bottom, p);
	        }

	        canvas.restoreToCount(saveCount);

	        // Step 6, draw decorations (scrollbars)
	        onDrawScrollBars(canvas);

	        if (mOverlay != null && !mOverlay.isEmpty()) {
	            mOverlay.getOverlayView().dispatchDraw(canvas);
	        }
	    }

    	上面一共分为绘制六部分
    			1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         由注释我们很容易看出绘制的哪一部分，我们主要来看下3，4部分


  	4）draw view's content   if (!dirtyOpaque) onDraw(canvas);

  		protected void onDraw(Canvas canvas) {
    	}

    	又是一个空的方法，那么具体实现应该在其子类里了，有子类来实现，这也就是为什么我们自定义view要重写onDraw方法了

   	5）draw the children dispatchDraw(canvas);

   		protected void dispatchDraw(Canvas canvas) {

    	}

    	也是一个空的方法，那么具体的实现也应该在其子类里了，因为只有ViewGroup容器有子类视图，所有应该是ViewGroup的实现了，下面我们来看下ViewGroup的实现

    	@Override
	    protected void dispatchDraw(Canvas canvas) {
	        boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
	        final int childrenCount = mChildrenCount;
	        final View[] children = mChildren;
	        int flags = mGroupFlags;

	        ...

	        int clipSaveCount = 0;
	        final boolean clipToPadding = (flags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK;
	        if (clipToPadding) {
	            clipSaveCount = canvas.save();
	            canvas.clipRect(mScrollX + mPaddingLeft, mScrollY + mPaddingTop,
	                    mScrollX + mRight - mLeft - mPaddingRight,
	                    mScrollY + mBottom - mTop - mPaddingBottom);
	        }

	        // We will draw our child's animation, let's reset the flag
	        mPrivateFlags &= ~PFLAG_DRAW_ANIMATION;
	        mGroupFlags &= ~FLAG_INVALIDATE_REQUIRED;

	        boolean more = false;
	        final long drawingTime = getDrawingTime();

	        if (usingRenderNodeProperties) canvas.insertReorderBarrier();
	        // Only use the preordered list if not HW accelerated, since the HW pipeline will do the
	        // draw reordering internally
	        final ArrayList<View> preorderedList = usingRenderNodeProperties
	                ? null : buildOrderedChildList();
	        final boolean customOrder = preorderedList == null
	                && isChildrenDrawingOrderEnabled();
	        //遍历所有的子视图
	        for (int i = 0; i < childrenCount; i++) {
	            int childIndex = customOrder ? getChildDrawingOrder(childrenCount, i) : i;
	            final View child = (preorderedList == null)
	                    ? children[childIndex] : preorderedList.get(childIndex);
	            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
	                more |= drawChild(canvas, child, drawingTime);
	            }
	        }
	        。。。
	    }

	    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        	return child.draw(canvas, this, drawingTime);
    	}

    	最后调用了view。draw方法，形成一个嵌套循环，把所有的view都绘制完成

四、surface上的工作
	

















