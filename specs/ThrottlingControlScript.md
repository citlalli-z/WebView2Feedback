Throttling Control - Script Throttling
===

# Background
Web content in WebView2 is generally subject to the same Web Platform
restrictions as in the Microsoft Edge browser. However, some of the scenarios
for WebView2 applications differ from the scenarios in the browser. For this
reason, we're providing a set of APIs to fine-tune performance of scripts
running in WebView2. These APIs allow WebView2 applications to achieve two
things:

* Customize script timers (`setTimeout` and `setInterval`) throttling under
different page states (foreground, background, and background with intensive
throttling)
* Throttle script timers in select hosted iframes

# Examples

## Throttle timers in visible WebView

Throttling Control APIs allow you to throttle JavaScript timers in scenarios
where the WebView2 control in your application needs to remain visible, but
consume less resources, for example, when the user is not interactive.

```c#
void OnNoUserInteraction()
{
    // User is not interactive, keep webview visible but throttle timers to 500ms.
    webView.CoreWebView2.SetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory.Foreground, 500);
}

void OnUserInteraction()
{
    // User is interactive again, unthrottle foreground timers.
    webView.CoreWebView2.SetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory.Foreground, 0);
}
```

```cpp
void ScenarioThrottlingControl::OnNoUserInteraction()
{
    auto webView21 = m_webview.try_query<ICoreWebView2_21>();
    CHECK_FEATURE_RETURN_EMPTY(webView21);

    // User is not interactive, keep webview visible but throttle timers to
    // 500ms.
    CHECK_FAILURE(webView21->SetThrottlingIntervalPreference(
        COREWEBVIEW2_THROTTLING_CATEGORY_FOREGROUND, 500));
}

void ScenarioThrottlingControl::OnUserInteraction()
{
    auto webView21 = m_webview.try_query<ICoreWebView2_21>();
    CHECK_FEATURE_RETURN_EMPTY(webView21);

    // User is interactive again, unthrottle foreground timers.
    CHECK_FAILURE(webView21->SetThrottlingIntervalPreference(
        COREWEBVIEW2_THROTTLING_CATEGORY_FOREGROUND, 0));
}
```

## Unthrottle timers in hidden WebView

Throttling Control APIs allow you to set a custom throttling interval for timers
on hidden WebViews. For example, if there's logic in your app that runs in
JavaScript but doesn't need to render content, you can keep the WebView hidden
and unthrottle its timers.

```C#
void SetupHiddenWebViewCore()
{
    // This WebView2 will remain hidden but needs to keep running timers.
    // Unthrottle background timers.
    webView.CoreWebView2.SetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory.Background, 0);
    // Effectively disable intensive throttling by overriding its timer interval.
    webView.CoreWebView2.SetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory.Intensive, 0);
    webView.Visibility = System.Windows.Visibility.Hidden;
}

void DisableHiddenWebViewCore()
{
    webView.Visibility = System.Windows.Visibility.Visible;
    webView.CoreWebView2.SetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory.Background, 1000);
    webView.CoreWebView2.SetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory.Intensive, 60000);
}
```

```cpp
void ScenarioThrottlingControl::SetupHiddenWebViewCore()
{
    auto webView21 = m_webview.try_query<ICoreWebView2_21>();
    CHECK_FEATURE_RETURN_EMPTY(webView21);

    // This WebView2 will remain hidden but needs to keep running timers.
    // Unthrottle background timers.
    CHECK_FAILURE(webView21->SetThrottlingIntervalPreference(
        COREWEBVIEW2_THROTTLING_CATEGORY_BACKGROUND, 0));
    // Effectively disable intensive throttling by overriding its timer interval.
    CHECK_FAILURE(webView21->SetThrottlingIntervalPreference(
        COREWEBVIEW2_THROTTLING_CATEGORY_INTENSIVE, 0));

    CHECK_FAILURE(m_appWindow->GetWebViewController()->put_IsVisible(FALSE));
}

void ScenarioThrottlingControl::DisableHiddenWebViewCore()
{
    CHECK_FAILURE(m_appWindow->GetWebViewController()->put_IsVisible(TRUE));

    auto webView21 = m_webview.try_query<ICoreWebView2_21>();
    CHECK_FEATURE_RETURN_EMPTY(webView21);

    CHECK_FAILURE(webView21->SetThrottlingIntervalPreference(
        COREWEBVIEW2_THROTTLING_CATEGORY_BACKGROUND, 1000));
    CHECK_FAILURE(webView21->SetThrottlingIntervalPreference(
        COREWEBVIEW2_THROTTLING_CATEGORY_INTENSIVE, 60000));
}
```

## Throttle timers in hosted iframes

Throttling Control APIs allow you to throttle timers in specific frames within
the WebView2 control. For example, if your application uses iframes to host 3rd
party content, you can select and mark these frames to be throttled separately
from the main frame and other regular, unmarked frames.

```C#
void SetupIsolatedFramesHandler()
{
    webView.CoreWebView2.FrameCreated += (sender, args) =>
    {
        // You can use the frame properties to determine whether it should be
        // marked to be throttled separately from main frame.
        if (args.Frame.Name == "isolated")
        {
            args.Frame.ShouldUseIsolatedThrottling = true;
        }
    };

    webView.CoreWebView2.SetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory.UntrustedFrame, 500);
}
```

```cpp
void ScenarioThrottlingControl::SetupIsolatedFramesHandler()
{
    auto webview4 = m_webview.try_query<ICoreWebView2_4>();
    CHECK_FEATURE_RETURN_EMPTY(webview4);

    // You can use the frame properties to determine whether it should be
    // marked to be throttled separately from main frame.
    CHECK_FAILURE(webview4->add_FrameCreated(
        Callback<ICoreWebView2FrameCreatedEventHandler>(
            [this](ICoreWebView2* sender, ICoreWebView2FrameCreatedEventArgs* args) -> HRESULT
            {
                wil::com_ptr<ICoreWebView2Frame> webviewFrame;
                CHECK_FAILURE(args->get_Frame(&webviewFrame));

                auto webviewFrame6 =
                    webviewFrame.try_query<ICoreWebView2Frame6>();
                CHECK_FEATURE_RETURN_HRESULT(webviewFrame6);

                wil::unique_cotaskmem_string name;
                CHECK_FAILURE(webviewFrame->get_Name(&name));
                if (wcscmp(name.get(), L"isolated") == 0)
                {
                    CHECK_FAILURE(webviewFrame6->put_ShouldUseIsolatedThrottling(TRUE));
                }

                return S_OK;
            })
            .Get(),
        &m_frameCreatedToken));

    auto webView21 = m_webview.try_query<ICoreWebView2_21>();
    CHECK_FAILURE(webView21->SetThrottlingIntervalPreference(
        COREWEBVIEW2_THROTTLING_CATEGORY_ISOLATED, 500));
}
```

# API Details
```cpp
[v1_enum]
typedef enum COREWEBVIEW2_THROTTLING_CATEGORY {
  /// Applies to frames whose WebView is in foreground state. WebViews whose
  /// `IsVisible` property is `TRUE` are in this state. The default value is a
  /// constant determined by the running version of the WebView2 Runtime.
  COREWEBVIEW2_THROTTLING_CATEGORY_FOREGROUND,

  /// Applies to frames whose WebView is in background state. WebViews whose
  /// `IsVisible` property is `FALSE` are in this state. The default value is a
  /// constant determined by the running version of the WebView2 Runtime.
  /// All other background state policies (including intensive throttling) are
  /// effective independently of this setting.
  COREWEBVIEW2_THROTTLING_CATEGORY_BACKGROUND,

  /// Applies to frames whose WebView is being intensively throttled (a
  /// sub-state of background state). For more details about intensive
  /// throttling, see [Intensive throttling of Javascript timer wake ups](https://chromestatus.com/feature/4718288976216064).
  /// The default value is a constant determined by the running version of the
  /// WebView2 Runtime.
  COREWEBVIEW2_THROTTLING_CATEGORY_INTENSIVE,

  /// Applies to frames that have been marked for isolated throttling by the host app.
  /// This is a category specific to WebView2 with no corresponding state in the
  /// Chromium tab state model. The default value is a constant determined by
  /// the running version of the WebView2 Runtime.
  COREWEBVIEW2_THROTTLING_CATEGORY_ISOLATED
} COREWEBVIEW2_THROTTLING_CATEGORY;

/// A continuation of the `ICoreWebView2` interface to support ThrottlingPreference.
[uuid(00f1b5fb-91ed-4722-9404-e0f8fd1e6b0a), object, pointer_default(unique)]
interface ICoreWebView2_21 : ICoreWebView2_20 {
  /// Get the preferred wake up interval (in milliseconds) for throttleable
  /// JavaScript tasks (`setTimeout` and `setInterval`), for the given
  /// throttling category. A wake up interval is the amount of time that needs
  /// to pass before the WebView2 Runtime checks for new timer tasks to run.
  /// The default interval values are constants determined by the running
  /// version of the WebView2 Runtime.
  HRESULT GetThrottlingIntervalPreference(
      [in] COREWEBVIEW2_THROTTLING_CATEGORY category,
      [out, retval] UINT32* intervalInMilliseconds);

  /// Sets the preferred wake up interval (in milliseconds) for throttleable
  /// JavaScript tasks (`setTimeout` and `setInterval`), for the given
  /// throttling category. A wake up interval is the amount of time that needs
  /// to pass before the WebView2 Runtime checks for new timer tasks to run. For
  /// example, an application might use a foreground value of 30 ms for moderate
  /// throttling scenarios, or match the default background value (usually 1000
  /// ms). The WebView2 Runtime will try to respect the preferred interval set
  /// by the application, but the effective value will be constrained by
  /// resource and platform limitations. Setting a value of `0` means a
  /// preference of no throttling to be applied.
  HRESULT SetThrottlingIntervalPreference(
      [in] COREWEBVIEW2_THROTTLING_CATEGORY category,
      [in] UINT32 intervalInMilliseconds);
}

/// A continuation of the `ICoreWebView2Frame` interface to support
/// ShouldUseIsolatedThrottling property.
[uuid(5b7d1b96-699b-44a2-b9f1-b8e88f9ac2be), object, pointer_default(unique)]
interface ICoreWebView2Frame6 : ICoreWebView2Frame5 {
  /// The `ShouldUseIsolatedThrottling` property indicates whether the frame has
  /// been marked for isolated throttling by the host app. Isolated throttling
  /// frames will receive a different script throttling category as compared to
  /// regular frames. Defaults to `FALSE` unless set otherwise. When `FALSE`,
  /// and for main frame, throttling category will be determined by page state.
  /// The corresponding preferred interval will apply (set through
  /// `SetThrottlingIntervalPreference`).
  [propget] HRESULT ShouldUseIsolatedThrottling([out, retval] BOOL* value);

  /// Marks the frame for isolated script throttling.
  [propput] HRESULT ShouldUseIsolatedThrottling([in] BOOL value);
}

```

```C#
namespace Microsoft.Web.WebView2.Core
{
    enum CoreWebView2ThrottlingCategory
    {
        Foreground = 0,
        Background = 1,
        Intensive = 2,
        UntrustedFrame = 3,
    };

    runtimeclass CoreWebView2
    {
        // ...

        [interface_name("Microsoft.Web.WebView2.Core.ICoreWebView2_21")]
        {
            UInt32 GetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory category);

            void SetThrottlingIntervalPreference(CoreWebView2ThrottlingCategory category, UInt32 intervalInMilliseconds);
        }
    }

    runtimeclass CoreWebView2Frame
    {
        // ...

        [interface_name("Microsoft.Web.WebView2.Core.ICoreWebView2Frame6")]
        {
            Boolean ShouldUseIsolatedThrottling { get; set; };
        }
    }
}

```

# Appendix