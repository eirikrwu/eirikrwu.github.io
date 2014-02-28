---

layout: post
title: "Windows屏幕亮度自动调节工具：autolight"

---

{% include JB/setup %}

最近买了个显示器，看网页写代码的时候和玩游戏看电影的时候对屏幕的亮度要求显著不同，手动调来调去太麻烦，所以写了个小工具自动调节屏幕亮度。代码放在[github](https://github.com/eirikrwu/autolight)上，在VisualStudio 2013和Windows 7下编译测试。

<!--more-->

这个工具基本上就是常驻后台，然后每隔一段时间检测一下当前用户是否在运行全屏程序，如果在全屏就认为用户在电影或游戏，就调高亮度，否则调低亮度。因此主要就是两个函数，第一个是检测用户是否在运行全屏程序：

{% highlight cpp %}
	bool IsFullScreenAppRunning() {
		HWND hWnd = GetForegroundWindow();
		RECT appBounds;
		RECT rc;
		GetWindowRect(GetDesktopWindow(), &rc);

		if (hWnd != GetDesktopWindow() && hWnd != GetShellWindow()) {
			GetWindowRect(hWnd, &appBounds);
			log_info("The bounds of the foreground window is %d x %d.", appBounds.right - appBounds.left, appBounds.bottom - appBounds.top);
			return appBounds.bottom >= rc.bottom && appBounds.left <= rc.left && appBounds.right >= rc.right && appBounds.top <= rc.top;
		} else {
			log_info("Foreground window is either desktop or shell.");
			return false;
		}
	}
{% endhighlight %}

用`GetForegroundWindow()`拿到当前窗口句柄，然后用`GetWindowRect`得到当前窗口的大小，再和桌面的大小做个比较就可以。这里需要注意的就是要去除当前窗口就是桌面本身的情况。

第二个是调节显示器亮度的函数，这里需要显示器支持DDC/CI，就是一个控制显示器参数的接口规范，具体的可以参考[MSDN关于Low-Level Monitor Configuration Functions][1]的说明。

{% highlight cpp %}
	bool SetBrightness(DWORD wTargetBrightness) {
		if (wTargetBrightness < 0) {
			return false;
		}

		// get forground window handle
		HWND hWnd = GetForegroundWindow();

		// get monitor handle
		HMONITOR hMonitor = MonitorFromWindow(hWnd, MONITOR_DEFAULTTONEAREST);

		// get how many physical monitors are associated with the handle
		DWORD dwPhysicalMonitorArraySize;
		if (!GetNumberOfPhysicalMonitorsFromHMONITOR(hMonitor, &dwPhysicalMonitorArraySize)) {
			return false;
		}

		// get physical monitors
		_PHYSICAL_MONITOR *pPhysicalMonitorArray = new _PHYSICAL_MONITOR[dwPhysicalMonitorArraySize];
		if (!GetPhysicalMonitorsFromHMONITOR(hMonitor, dwPhysicalMonitorArraySize, pPhysicalMonitorArray)) {
			return false;
		}

		// for each physical monitor
		for (int i = 0; i < dwPhysicalMonitorArraySize; i++) {
			// get physical monitor handle
			HANDLE hPhysicalMonitor = pPhysicalMonitorArray[i].hPhysicalMonitor;

			// get the current brightness
			MC_VCP_CODE_TYPE pvct;
			DWORD wCurrentValue;
			DWORD wMaximumValue;
			if (!GetVCPFeatureAndVCPFeatureReply(hPhysicalMonitor, VCP_CODE_BRIGHTNESS, &pvct, &wCurrentValue, &wMaximumValue)) {
				continue;
			}

			// check if adjustment is needed
			if (wTargetBrightness != wCurrentValue && wTargetBrightness <= wMaximumValue) {
				if (!SetVCPFeature(hPhysicalMonitor, VCP_CODE_BRIGHTNESS, wTargetBrightness)) {
					// log failure
					continue;
				}
			}
		}

		delete[] pPhysicalMonitorArray;

		return true;
	}
{% endhighlight %}

接下来就简单了，写一个while true循环每`Sleep(60000)`检测一下是否全屏并相应的调节亮度就ok。

编译好的可执行文件放在github repo的executable文件夹下，支持三个可选参数，分别是低亮度值，高亮度值，检测间隔时间。例如：

	autolight 20 80 30

表示低亮度20，高亮度80，每隔30秒检测一次用户是否在运行全屏程序。三个参数的默认值分别是25,75,60。

[1]: http://msdn.microsoft.com/en-us/library/dd692982(v=vs.85).aspx