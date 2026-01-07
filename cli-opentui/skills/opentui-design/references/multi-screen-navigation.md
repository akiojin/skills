# Multi-Screen Navigation in OpenTUI

Patterns for implementing screen navigation in OpenTUI CLI applications.

## Overview

CLI applications often need multiple screens (views) that users navigate between. This guide covers patterns for managing screen transitions safely.

## Basic Screen Manager

```typescript
import { createSignal, Show } from "solid-js";

type Screen = "main" | "detail" | "settings";

const App = () => {
  const [currentScreen, setCurrentScreen] = createSignal<Screen>("main");

  return (
    <box flexDirection="column" flexGrow={1}>
      <Show when={currentScreen() === "main"}>
        <MainScreen onNavigate={setCurrentScreen} />
      </Show>
      <Show when={currentScreen() === "detail"}>
        <DetailScreen onNavigate={setCurrentScreen} />
      </Show>
      <Show when={currentScreen() === "settings"}>
        <SettingsScreen onNavigate={setCurrentScreen} />
      </Show>
    </box>
  );
};
```

## Screen Stack Pattern

For back navigation support.

```typescript
const useScreenStack = () => {
  const [stack, setStack] = createSignal<Screen[]>(["main"]);

  const push = (screen: Screen) => {
    setStack((prev) => [...prev, screen]);
  };

  const pop = () => {
    setStack((prev) => (prev.length > 1 ? prev.slice(0, -1) : prev));
  };

  const current = () => stack()[stack().length - 1];
  const canGoBack = () => stack().length > 1;

  return { current, push, pop, canGoBack };
};

const App = () => {
  const { current, push, pop, canGoBack } = useScreenStack();

  useKeyboard((key) => {
    if (key.name === "escape" && canGoBack()) {
      pop();
    }
  });

  return (
    <Switch>
      <Match when={current() === "main"}>
        <MainScreen onNavigate={push} />
      </Match>
      <Match when={current() === "detail"}>
        <DetailScreen onBack={pop} />
      </Match>
    </Switch>
  );
};
```

## Preventing Key Propagation on Transition

Critical pattern to prevent Enter key from firing on newly mounted screens.

### Source Screen

```typescript
const MainScreen = (props: { onNavigate: (screen: Screen) => void }) => {
  useKeyboard((key) => {
    if (key.name === "return") {
      key.preventDefault(); // CRITICAL: Prevent propagation
      props.onNavigate("detail");
    }
  });

  return <box>Main Screen</box>;
};
```

### Destination Screen

```typescript
const DetailScreen = (props: { onBack: () => void }) => {
  const [ready, setReady] = createSignal(false);

  onMount(() => {
    // Delay input acceptance
    const timer = setTimeout(() => setReady(true), 150);
    return () => clearTimeout(timer);
  });

  useKeyboard((key) => {
    if (!ready()) return; // Ignore input during delay

    if (key.name === "return") {
      // Safe to handle Enter
    }

    if (key.name === "escape") {
      props.onBack();
    }
  });

  return <box>Detail Screen</box>;
};
```

## Screen with Parameters

Pass data between screens.

```typescript
interface ScreenParams {
  main: undefined;
  detail: { itemId: string };
  edit: { itemId: string; mode: "create" | "update" };
}

type ScreenName = keyof ScreenParams;

interface NavigationState {
  screen: ScreenName;
  params?: ScreenParams[ScreenName];
}

const useNavigation = () => {
  const [state, setState] = createSignal<NavigationState>({
    screen: "main",
  });

  const navigate = <T extends ScreenName>(
    screen: T,
    params?: ScreenParams[T]
  ) => {
    setState({ screen, params });
  };

  return { state, navigate };
};

// Usage
const App = () => {
  const { state, navigate } = useNavigation();

  return (
    <Switch>
      <Match when={state().screen === "main"}>
        <MainScreen
          onSelectItem={(id) => navigate("detail", { itemId: id })}
        />
      </Match>
      <Match when={state().screen === "detail"}>
        <DetailScreen
          itemId={(state().params as { itemId: string }).itemId}
          onBack={() => navigate("main")}
        />
      </Match>
    </Switch>
  );
};
```

## Modal Dialogs

Overlay screens that block underlying input.

```typescript
const App = () => {
  const [showModal, setShowModal] = createSignal(false);

  return (
    <box flexDirection="column" flexGrow={1}>
      <MainScreen
        active={!showModal()}
        onShowModal={() => setShowModal(true)}
      />

      <Show when={showModal()}>
        <Modal onClose={() => setShowModal(false)}>
          <text>Modal content</text>
        </Modal>
      </Show>
    </box>
  );
};

const Modal = (props: { onClose: () => void; children: JSX.Element }) => {
  useKeyboard((key) => {
    if (key.name === "escape") {
      props.onClose();
    }
    // Consume all keys to block underlying components
    key.preventDefault();
  });

  return (
    <box
      position="absolute"
      top="center"
      left="center"
      width="80%"
      height="50%"
      borderStyle="double"
    >
      {props.children}
    </box>
  );
};
```

## Wizard/Multi-Step Flow

Sequential screens with state preservation.

```typescript
interface WizardState {
  step: number;
  data: {
    name?: string;
    email?: string;
    confirmed?: boolean;
  };
}

const Wizard = () => {
  const [state, setState] = createStore<WizardState>({
    step: 0,
    data: {},
  });

  const nextStep = () => setState("step", (s) => s + 1);
  const prevStep = () => setState("step", (s) => Math.max(0, s - 1));
  const updateData = (key: string, value: unknown) =>
    setState("data", key, value);

  return (
    <box flexDirection="column">
      <Switch>
        <Match when={state.step === 0}>
          <NameStep
            value={state.data.name}
            onChange={(v) => updateData("name", v)}
            onNext={nextStep}
          />
        </Match>
        <Match when={state.step === 1}>
          <EmailStep
            value={state.data.email}
            onChange={(v) => updateData("email", v)}
            onNext={nextStep}
            onBack={prevStep}
          />
        </Match>
        <Match when={state.step === 2}>
          <ConfirmStep data={state.data} onBack={prevStep} onSubmit={submit} />
        </Match>
      </Switch>

      <ProgressIndicator current={state.step} total={3} />
    </box>
  );
};
```

## Transition Animations

Simple fade-like transitions using loading state.

```typescript
const useScreenTransition = (delay = 100) => {
  const [transitioning, setTransitioning] = createSignal(false);
  const [currentScreen, setCurrentScreen] = createSignal<Screen>("main");

  const navigate = (screen: Screen) => {
    setTransitioning(true);
    setTimeout(() => {
      setCurrentScreen(screen);
      setTransitioning(false);
    }, delay);
  };

  return { currentScreen, navigate, transitioning };
};

const App = () => {
  const { currentScreen, navigate, transitioning } = useScreenTransition();

  return (
    <box>
      <Show when={transitioning()}>
        <text dimColor>Loading...</text>
      </Show>
      <Show when={!transitioning()}>
        <Dynamic
          component={screens[currentScreen()]}
          onNavigate={navigate}
        />
      </Show>
    </box>
  );
};
```

## Best Practices

1. **Always use preventDefault()** - When navigating with Enter key
2. **Add input delay on mount** - 100-150ms buffer for new screens
3. **Support Escape for back** - Consistent navigation pattern
4. **Show current location** - Breadcrumbs or title
5. **Preserve state when needed** - Use stack or context
6. **Handle edge cases** - Empty stack, invalid params
7. **Disable underlying input** - When modal is open
