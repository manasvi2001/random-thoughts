### **📌 Summary: Scalable Next.js Architecture with Widget-Based Rendering & Location-Based Data Fetching**

---

Since your **Next.js project has grown large**, your** suggestion was a "Page → Screen → Container → Item"** structure, but this felt **more aligned with React Native** rather than a web-based project.

After discussing different approaches, we considered:

1. **Scalable Architectural Patterns** for structuring the project.
2. **Handling Dynamic Widget-Based Rendering**, where widgets are dynamically rendered from backend API data.
3. **Efficiently Fetching Data Based on User Location** while handling permission states.

This summary **details the considerations, trade-offs, and the most recommended way to structure and optimize your project**.

---

## **1️⃣ Architectural Approaches Considered**

To **structure the project efficiently**, we explored multiple architecture styles:

### **🔹 (A) Feature-Based Architecture**

Each feature has **its own folder**, grouping everything related to that feature.

📂 **Example:**

```
/src
  /features
    /dashboard
      DashboardPage.tsx
      DashboardContainer.tsx
      DashboardScreen.tsx
      components/
        DashboardHeader.tsx
        DashboardStats.tsx
      widgets/
        WidgetRenderer.tsx
      hooks/
        useDashboard.ts
      services/
        dashboardService.ts
  /components (🔹 Reusable UI components)
  /pages (🔹 Next.js routing)
```

✅ **Pros**:

- Independent features make the project **modular**.
- **Easy to scale** as new features are added.
- **Encourages separation of concerns**.

❌ **Cons**:

- Some duplication if different features require similar UI components.

### **🔹 (B) Atomic Design for UI Consistency**

Components are structured as:

- **Atoms** (e.g., Button, Input)
- **Molecules** (e.g., SearchBar, ProfileCard)
- **Organisms** (e.g., Header, Sidebar)

📂 **Example:**

```
/components
  /atoms
    Button.tsx
    Input.tsx
  /molecules
    SearchBar.tsx
  /organisms
    Header.tsx
```

✅ **Pros**:

- Encourages **reusable UI components**.
- **Improves design consistency** across the app.

❌ **Cons**:

- **Does not handle business logic** (needs to be combined with Feature-Based Architecture).

### **🔹 (C) Container-Presenter Pattern**

Separates **UI (presenters)** and **business logic (containers)**.

📂 **Example:**

```
/components
  /presentational (🔹 UI components)
    Button.tsx
    Card.tsx
  /containers (🔹 Data & business logic)
    DashboardContainer.tsx
```

✅ **Pros**:

- **Great for testability** (pure UI components).
- **Better separation of concerns**.

❌ **Cons**:

- Adds **boilerplate code**.
- Can be **overkill** for some cases.

### **🔹 (D) Domain-Driven Design (DDD)**

Used in **complex enterprise applications**.

📂 **Example:**

```
/domains
  /user
    User.ts
    UserService.ts
    UserRepository.ts
```

✅ **Pros**:

- **Scalable for complex business logic**.
- **Encourages clear domain boundaries**.

❌ **Cons**:

- **Overkill for most UI-heavy projects**.

---

## **2️⃣ Dynamic Widget-Based Rendering**

Your project includes **widget-based data**, where the backend determines what widgets are rendered.

✅ **Key Considerations**:

- **Widgets should be modular & reusable**.
- **Dynamically fetch and render widgets** based on API response.
- **Optimize performance using lazy loading**.

📂 **Minimal Change to Your Existing Structure:**

```
/widgets
  WidgetRenderer.tsx
  widgets/
    ChartWidget.tsx
    TableWidget.tsx
    TextWidget.tsx
```

### **🔹 Widget Renderer Logic**

Instead of inline `switch-case` statements in components:

```tsx
import dynamic from 'next/dynamic';

const ChartWidget = dynamic(() => import('./widgets/ChartWidget'));
const TableWidget = dynamic(() => import('./widgets/TableWidget'));
const TextWidget = dynamic(() => import('./widgets/TextWidget'));

const widgetMapping = {
  chart: ChartWidget,
  table: TableWidget,
  text: TextWidget,
};

const WidgetRenderer = ({ widget }) => {
  const Component = widgetMapping[widget.type];
  if (!Component) return <div>Unknown Widget</div>;

  return <Component data={widget.data} />;
};

export default WidgetRenderer;
```

✅ **Pros**:

- **Easier to maintain**.
- **New widgets can be added dynamically**.

---

## **3️⃣ Optimized Location-Based Data Fetching**

Your application **fetches widgets only after obtaining user location**.\
Challenges:

- Handling **permissions** (`granted`, `denied`, `pending`).
- **Delaying API calls** until location is available.
- **Caching last known location**.

### **🔹 ****`useUserLocation`**** Hook (Location Handling)**

```tsx
import { useEffect, useState } from 'react';

const getStoredLocation = () => {
  if (typeof window !== 'undefined') {
    const stored = localStorage.getItem('userLocation');
    return stored ? JSON.parse(stored) : null;
  }
  return null;
};

export const useUserLocation = () => {
  const [location, setLocation] = useState(null);
  const [status, setStatus] = useState('pending');

  useEffect(() => {
    const storedLocation = getStoredLocation();
    if (storedLocation) setLocation(storedLocation);

    if (navigator.geolocation) {
      navigator.permissions.query({ name: 'geolocation' }).then((result) => {
        if (result.state === 'granted') {
          setStatus('granted');
          navigator.geolocation.getCurrentPosition(
            (pos) => {
              const newLocation = { latitude: pos.coords.latitude, longitude: pos.coords.longitude };
              setLocation(newLocation);
              localStorage.setItem('userLocation', JSON.stringify(newLocation));
            },
            () => setStatus('denied')
          );
        } else if (result.state === 'denied') {
          setStatus('denied');
        }
      });
    } else {
      setStatus('denied');
    }
  }, []);

  return { location, status };
};
```

### **🔹 ****`useWidgets`**** Hook (Fetch Data After Location)**

```tsx
import { useEffect, useState } from 'react';
import { useUserLocation } from './useUserLocation';

const fetchWidgets = async (latitude, longitude) => {
  const response = await fetch(`/api/widgets?lat=${latitude}&lng=${longitude}`);
  return response.json();
};

export const useWidgets = () => {
  const { location, status } = useUserLocation();
  const [widgets, setWidgets] = useState([]);

  useEffect(() => {
    if (status === 'granted' && location) {
      fetchWidgets(location.latitude, location.longitude).then(setWidgets);
    }
  }, [location, status]);

  return { widgets, isLoading: !widgets.length && status !== 'denied' };
};
```

✅ **Pros**:

- **Ensures API calls only happen after location is available**.
- **Caches last known location** to improve performance.

---

## **🎯 Final Recommendation: Best Approach for Your Case**

Since a **full project restructure is not feasible**, the best **incremental improvement** strategy is:

1. **Keep the current structure but organize widgets into ****`/widgets`**.
2. **Use a centralized ****`WidgetRenderer.tsx`** to handle widget logic.
3. **Introduce ****`useUserLocation`**** & ****`useWidgets`**** hooks** to manage location-based API calls.
4. **Lazy-load widgets for performance optimization**.

✅ **Pros of this approach**:
✔ **Minimal code changes**\
✔ **Easier maintenance**\
✔ **Optimized API calls**\
✔ **Improved performance (lazy loading, caching)**

---

Would you like a **migration plan** for refactoring your project in **small steps**? 🚀
