---
layout: post
title: "view and ViewGroup's measure layout and draw"
date: 2015-05-19 16:35:34 +0800
comments: true
categories: android
---

- Begin from ViewRoot's performTrasversals() -> performMeasure() -> performLayout() -> performDraw()
- Measure: pass measurespec to all children and ask them to measure themselves base on the measurespec value  

	<!--more-->

 	- Measure is a traversal procedure, from parent to child.
 	- MeasureSpec is a integer conbination of a size and a mode:
 		- size: size provided by parent to calculate measured size
 		- mode: limit provided by parent to calculate measured size, possible values as below:
 			- AT_MOST: the measured size can not be larger than specified size.
 			- EXACTLY: the measured size must be equal to specified size.
 			- UNSPECIFIED: the child view can be whatever size it wants.  
 		- Why setting width/height to 0dp when child view in LinearLayout has weight?
 		- getMeasuredWidth/height will always return 0 until measure finish.
 - Layout: Based on the result of measure, now we know the child views's approximate dimensions, the next procedure is decide the location of a view. Just like drawing a rect on a white paper, we must know the size of this rect, the distance between the rect's edge and the pager's edge, then we can draw it down. The layout procedure need four parameters:
 	- left
	- top
	- right
	- bottom
	
	These four parameters indicate the distance between view edge and parent's left top point.  
	
	In View's onLayout(), do nothing.  
	
	In View's layout(), call setFrame(), then mLeft, mRight, mTop, mBottom is assigned value, then getWidth()/getHeigth() will not return 0 but the correct value. That is why we can not call getWidth()/getHeight() in activity's onCreate() method, but in view.getViewTreeObserver().addOnGlobalLayoutListener(), because only then the layout procedure is finished.  
	
	In layout procedure, we can also decide the size of a view, by changing the left/top/right/bottom value, but generally, we don't do this.  
	
- Draw: after measure and layout, the exactly visble area of the view is determined, now we will draw the view/viewgroup.  

	When drawing, don't determine drawing location base on canvas.getWidth()/getHeight(), the canvas size won't be equal to the view size, use view.getWidth()/getHeight() instead(On android 2.3, the canvas size will be equal to the screen size in some case).  
	
	In View's onDraw(), do nothing, you can define you own drawing mechanism here.  
	
	In View's draw(), draw background -> draw content(onDraw) -> draw children(dispatchDraw)
	
	Draw children by calling dispatchDraw(), in View's dispatchDraw(), do nothing.  
	
	In ViewGroup's dispatchDraw(), use a for loop to call child's draw() method. If clipToPadding is set to true(by default is true), the canvas pass to child will be clip base on ViewGroup' size and scrolling position. That is why children can not draw outside its parent if not call setClipToPadding(false).  
	
	In View's source code, there is two draw() method, draw(Canvas) and draw(Canvas, ViewGroup, long), the second one is call by parent ViewGroup's dispatchDraw(), and in this method, handle with the canvas(clip, translate and so on) and call the draw(Canvas).
- measure() -> onMeasure()
- layout() -> onLayout()
- draw() -> onDraw()
- Do not override xxx(), override onXXX() instead.
- ViewGroup 's onLayout is abstract.
- ViewGroup will not call onDraw until you call setWillNotDraw(false).
- measure/layout/draw spend time < 16ms -> butter
- measure/layout/draw spend time > 16ms -> janky
- Improve performance:
	- measure: FrameLayout -> RelativeLayout -> LinearLayout
	- layout: reduce the number of layout level.
		- ImageView + TextView -> TextView
		- Multi LinearLayout -> RelativeLayout
		- ...
	- draw: in customView, try your best to reduce draw time(overdraw), clip canvas and just draw the necessary area.
	- If necessary, you can try to draw content instead of using different view to make your UI(bypass measure layout, but will make coding harder). 