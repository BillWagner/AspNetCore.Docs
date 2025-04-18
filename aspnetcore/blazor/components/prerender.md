---
title: Prerender ASP.NET Core Razor components
author: guardrex
description: Learn about Razor component prerendering in ASP.NET Core Blazor apps.
monikerRange: '>= aspnetcore-8.0'
ms.author: riande
ms.custom: mvc
ms.date: 11/12/2024
uid: blazor/components/prerender
---
# Prerender ASP.NET Core Razor components

[!INCLUDE[](~/includes/not-latest-version-without-not-supported-content.md)]

<!--
    NOTE: The console output block quotes in this topic use a double-space 
    at the ends of lines to generate a bare return in block quote output.
-->

This article explains Razor component prerendering scenarios for server-rendered components in Blazor Web Apps.

*Prerendering* is the process of initially rendering page content on the server without enabling event handlers for rendered controls. The server outputs the HTML UI of the page as soon as possible in response to the initial request, which makes the app feel more responsive to users. Prerendering can also improve [Search Engine Optimization (SEO)](https://developer.mozilla.org/docs/Glossary/SEO) by rendering content for the initial HTTP response that search engines use to calculate page rank.

## Persist prerendered state

Without persisting prerendered state, state used during prerendering is lost and must be recreated when the app is fully loaded. If any state is created asynchronously, the UI may flicker as the prerendered UI is replaced when the component is rerendered.

Consider the following `PrerenderedCounter1` counter component. The component sets an initial random counter value during prerendering in [`OnInitialized` lifecycle method](xref:blazor/components/lifecycle#component-initialization-oninitializedasync). After the SignalR connection to the client is established, the component rerenders, and the initial count value is replaced when `OnInitialized` executes a second time.

`PrerenderedCounter1.razor`:

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/PrerenderedCounter1.razor":::

Run the app and inspect logging from the component. The following is example output.

> [!NOTE]
> If the app adopts [interactive routing](xref:blazor/fundamentals/routing#static-versus-interactive-routing) and the page is reached via an internal [enhanced navigation](xref:blazor/fundamentals/routing#enhanced-navigation-and-form-handling), prerendering doesn't occur. Therefore, you must perform a full page reload for the `PrerenderedCounter1` component to see the following output. For more information, see the [Interactive routing and prerendering](#interactive-routing-and-prerendering) section.

> :::no-loc text="info: BlazorSample.Components.Pages.PrerenderedCounter1[0]":::  
> :::no-loc text="      currentCount set to 41":::  
> :::no-loc text="info: BlazorSample.Components.Pages.PrerenderedCounter1[0]":::  
> :::no-loc text="      currentCount set to 92":::

The first logged count occurs during prerendering. The count is set again after prerendering when the component is rerendered. There's also a flicker in the UI when the count updates from 41 to 92.

To retain the initial value of the counter during prerendering, Blazor supports persisting state in a prerendered page using the <xref:Microsoft.AspNetCore.Components.PersistentComponentState> service (and for components embedded into pages or views of Razor Pages or MVC apps, the [Persist Component State Tag Helper](xref:mvc/views/tag-helpers/builtin-th/persist-component-state-tag-helper)).

To preserve prerendered state, decide what state to persist using the <xref:Microsoft.AspNetCore.Components.PersistentComponentState> service. <xref:Microsoft.AspNetCore.Components.PersistentComponentState.RegisterOnPersisting%2A?displayProperty=nameWithType> registers a callback to persist the component state before the app is paused. The state is retrieved when the app resumes. Make the call at the end of initialization code in order to avoid a potential race condition during app shutdown.

The following example demonstrates the general pattern:

* The `{TYPE}` placeholder represents the type of data to persist.
* The `{TOKEN}` placeholder is a state identifier string. Consider using `nameof({VARIABLE})`, where the `{VARIABLE}` placeholder is the name of the variable that holds the state. Using [`nameof()`](/dotnet/csharp/language-reference/operators/nameof) for the state identifier avoids the use of a quoted string.

```razor
@implements IDisposable
@inject PersistentComponentState ApplicationState

...

@code {
    private {TYPE} data;
    private PersistingComponentStateSubscription persistingSubscription;

    protected override async Task OnInitializedAsync()
    {
        if (!ApplicationState.TryTakeFromJson<{TYPE}>(
            "{TOKEN}", out var restored))
        {
            data = await ...;
        }
        else
        {
            data = restored!;
        }

        // Call at the end to avoid a potential race condition at app shutdown
        persistingSubscription = ApplicationState.RegisterOnPersisting(PersistData);
    }

    private Task PersistData()
    {
        ApplicationState.PersistAsJson("{TOKEN}", data);

        return Task.CompletedTask;
    }

    void IDisposable.Dispose()
    {
        persistingSubscription.Dispose();
    }
}
```

The following counter component example persists counter state during prerendering and retrieves the state to initialize the component.

`PrerenderedCounter2.razor`:

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/PrerenderedCounter2.razor":::

When the component executes, `currentCount` is only set once during prerendering. The value is restored when the component is rerendered. The following is example output.

> [!NOTE]
> If the app adopts [interactive routing](xref:blazor/fundamentals/routing#static-versus-interactive-routing) and the page is reached via an internal [enhanced navigation](xref:blazor/fundamentals/routing#enhanced-navigation-and-form-handling), prerendering doesn't occur. Therefore, you must perform a full page reload for the `PrerenderedCounter2` component to see the following output. For more information, see the [Interactive routing and prerendering](#interactive-routing-and-prerendering) section.

> :::no-loc text="info: BlazorSample.Components.Pages.PrerenderedCounter2[0]":::  
> :::no-loc text="      currentCount set to 96":::  
> :::no-loc text="info: BlazorSample.Components.Pages.PrerenderedCounter2[0]":::  
> :::no-loc text="      currentCount restored to 96":::

By initializing components with the same state used during prerendering, any expensive initialization steps are only executed once. The rendered UI also matches the prerendered UI, so no flicker occurs in the browser.

The persisted prerendered state is transferred to the client, where it's used to restore the component state. During client-side rendering (CSR, `InteractiveWebAssembly`), the data is exposed to the browser and must not contain sensitive, private information. During interactive server-side rendering (interactive SSR, `InteractiveServer`), [ASP.NET Core Data Protection](xref:security/data-protection/introduction) ensures that the data is transferred securely. The `InteractiveAuto` render mode combines WebAssembly and Server interactivity, so it's necessary to consider data exposure to the browser, as in the CSR case.

## Components embedded into pages and views (Razor Pages/MVC)

For components embedded into a page or view of a Razor Pages or MVC app, you must add the [Persist Component State Tag Helper](xref:mvc/views/tag-helpers/builtin-th/persist-component-state-tag-helper) with the `<persist-component-state />` HTML tag inside the closing `</body>` tag of the app's layout. **This is only required for Razor Pages and MVC apps.** For more information, see <xref:mvc/views/tag-helpers/builtin-th/persist-component-state-tag-helper>.

`Pages/Shared/_Layout.cshtml`:

```cshtml
<body>
    ...

    <persist-component-state />
</body>
```

## Interactive routing and prerendering

When the `Routes` component doesn't define a render mode, the app is using per-page/component interactivity and navigation. Using per-page/component navigation, internal&dagger; navigation is handled by [enhanced routing](xref:blazor/fundamentals/routing#enhanced-navigation-and-form-handling) after the app becomes interactive. &dagger;*Internal* in this context means that the URL destination of the navigation event is a Blazor endpoint inside the app.

The <xref:Microsoft.AspNetCore.Components.PersistentComponentState> service only works on the initial page load and not across internal enhanced page navigation events.

If the app performs a full (non-enhanced) navigation to a page utilizing persistent component state, the persisted state is made available for the app to use when it becomes interactive.

If an interactive circuit has already been established and an enhanced navigation is performed to a page utilizing persistent component state, the state *isn't made available in the existing circuit for the component to use*. There's no prerendering for the internal page request, and the <xref:Microsoft.AspNetCore.Components.PersistentComponentState> service isn't aware that an enhanced navigation has occurred. There's no mechanism to deliver state updates to components that are already running on an existing circuit. The reason for this is that Blazor only supports passing state from the server to the client at the time the runtime initializes, not after the runtime has started.

Additional work on the Blazor framework to address this scenario is under consideration for .NET 10 (November, 2025). For more information and community discussion of *unsupported workarounds*&Dagger;, see [Support persistent component state across enhanced page navigations (`dotnet/aspnetcore` #51584)](https://github.com/dotnet/aspnetcore/issues/51584). &Dagger;Unsupported workarounds aren't sanctioned by Microsoft for use in Blazor apps. *Use third-party packages, approaches, and code at your own risk.*

Disabling enhanced navigation, which reduces performance but also avoids the problem of loading state with <xref:Microsoft.AspNetCore.Components.PersistentComponentState> for internal page requests, is covered in <xref:blazor/fundamentals/routing#enhanced-navigation-and-form-handling>.

<!-- UPDATE 10.0 The PU is probably going to address this
                 via the issue mentioned above. If so, 
                 the PU issue link above and the NOTEs for
                 the article examples will be versioned out 
                 for 10.0. -->

## Prerendering guidance

Prerendering guidance is organized in the Blazor documentation by subject matter. The following links cover all of the prerendering guidance throughout the documentation set by subject:

* Fundamentals
  * <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> is executed *twice* when prerendering: [Handle asynchronous navigation events with `OnNavigateAsync`](xref:blazor/fundamentals/routing#handle-asynchronous-navigation-events-with-onnavigateasync)
  * [Startup: Control headers in C# code](xref:blazor/fundamentals/startup#control-headers-in-c-code)
  * [Handle Errors: Prerendering](xref:blazor/fundamentals/handle-errors#prerendering)
  * [SignalR: Prerendered state size and SignalR message size limit](xref:blazor/fundamentals/signalr#prerendered-state-size-and-signalr-message-size-limit)

* [Render modes: Prerendering](xref:blazor/components/render-modes#prerendering)

* Components
  * [Control `<head>` content during prerendering](xref:blazor/components/control-head-content#control-head-content-during-prerendering)
  * Razor component lifecycle subjects that pertain to prerendering
    * [Component initialization (`OnInitialized{Async}`)](xref:blazor/components/lifecycle#component-initialization-oninitializedasync)
    * [After component render (`OnAfterRender{Async}`)](xref:blazor/components/lifecycle#after-component-render-onafterrenderasync)
    * [Stateful reconnection after prerendering](xref:blazor/components/lifecycle#stateful-reconnection-after-prerendering)
    * [Prerendering with JavaScript interop](xref:blazor/components/lifecycle#prerendering-with-javascript-interop): This section also appears in the two JS interop articles on calling JavaScript from .NET and calling .NET from JavaScript.
    * [Handle incomplete asynchronous actions at render](xref:blazor/components/lifecycle#handle-incomplete-asynchronous-actions-at-render): Guidance for delayed rendering due to long-running lifecycle tasks during prerendering on the server.
  * [`QuickGrid` component sample app](xref:blazor/components/quickgrid#sample-app): The [**QuickGrid for Blazor** sample app](https://aspnet.github.io/quickgridsamples/) is hosted on GitHub Pages. The site loads fast thanks to static prerendering using the community-maintained [`BlazorWasmPrerendering.Build` GitHub project](https://github.com/jsakamoto/BlazorWasmPreRendering.Build).
  * [Prerendering when integrating components into Razor Pages and MVC apps](xref:blazor/components/integration)

* Authentication and authorization
  * [Server-side threat mitigation: Cross-site scripting (XSS)](xref:blazor/security/interactive-server-side-rendering#cross-site-scripting-xss)
  * [Server-side unauthorized content display while prerendering with a custom `AuthenticationStateProvider`](xref:blazor/security/index#unauthorized-content-display-while-prerendering-with-a-custom-authenticationstateprovider)
  * [Blazor WebAssembly rendered component authentication with prerendering](xref:blazor/security/webassembly/additional-scenarios#prerendering-with-authentication)

* [State management: Handle prerendering](xref:blazor/state-management#handle-prerendering): Besides the *Handle prerendering* section, several of the article's other sections include remarks on prerendering.
