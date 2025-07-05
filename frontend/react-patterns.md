# Advanced React Design Patterns

Design patterns in React aren't just about writing cleaner code—they're about building applications that scale gracefully as your team grows and requirements evolve. After working on several large-scale React applications, I've found that the right patterns can make the difference between a maintainable codebase and technical debt that compounds over time.

This guide covers patterns that have proven their worth in production environments, with real-world trade-offs and practical implementation strategies.

## Container-Presenter Pattern

The Container-Presenter pattern (also known as Smart/Dumb components) separates data fetching and business logic from presentation concerns. This pattern shines when you need to reuse presentation logic across different data sources or when testing becomes complex.

### Why It Matters

In my experience with multilingual dashboards, this pattern was crucial for maintaining consistent UI behavior while handling different API endpoints for each language. The presenter components remained stable while containers handled the data orchestration.

```tsx
// Container Component
const UserListContainer = () => {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUsers = async () => {
      setLoading(true);
      try {
        const response = await api.getUsers();
        setUsers(response.data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };
    
    fetchUsers();
  }, []);

  const handleUserDelete = async (userId) => {
    try {
      await api.deleteUser(userId);
      setUsers(users.filter(user => user.id !== userId));
    } catch (err) {
      setError(err.message);
    }
  };

  return (
    <UserList
      users={users}
      loading={loading}
      error={error}
      onDelete={handleUserDelete}
    />
  );
};

// Presenter Component
const UserList = ({ users, loading, error, onDelete }) => {
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;

  return (
    <div className="user-list">
      {users.map(user => (
        <UserCard
          key={user.id}
          user={user}
          onDelete={() => onDelete(user.id)}
        />
      ))}
    </div>
  );
};
```

**Trade-offs:** While this pattern improves testability and reusability, it can lead to prop drilling in deep component trees. Consider using Context or state management libraries for complex scenarios.

## Compound Component Pattern

Compound components provide a flexible API that allows parent components to control the rendering and behavior of child components while maintaining a cohesive interface.

### Real-World Application

I've used this pattern extensively for form builders and data tables where you need fine-grained control over individual parts while maintaining a unified API.

```tsx
const DataTable = ({ children, data }) => {
  const [sortConfig, setSortConfig] = useState({ key: null, direction: 'asc' });
  const [filters, setFilters] = useState({});

  const sortedData = useMemo(() => {
    if (!sortConfig.key) return data;
    
    return [...data].sort((a, b) => {
      const aVal = a[sortConfig.key];
      const bVal = b[sortConfig.key];
      
      if (aVal < bVal) return sortConfig.direction === 'asc' ? -1 : 1;
      if (aVal > bVal) return sortConfig.direction === 'asc' ? 1 : -1;
      return 0;
    });
  }, [data, sortConfig]);

  return (
    <DataTableContext.Provider value={{ sortedData, sortConfig, setSortConfig, filters, setFilters }}>
      <table className="data-table">
        {children}
      </table>
    </DataTableContext.Provider>
  );
};

DataTable.Header = ({ children }) => (
  <thead>
    <tr>{children}</tr>
  </thead>
);

DataTable.Column = ({ field, children, sortable = false }) => {
  const { sortConfig, setSortConfig } = useContext(DataTableContext);
  
  const handleSort = () => {
    if (!sortable) return;
    
    setSortConfig(prev => ({
      key: field,
      direction: prev.key === field && prev.direction === 'asc' ? 'desc' : 'asc'
    }));
  };

  return (
    <th onClick={handleSort} className={sortable ? 'sortable' : ''}>
      {children}
      {sortable && sortConfig.key === field && (
        <span>{sortConfig.direction === 'asc' ? '↑' : '↓'}</span>
      )}
    </th>
  );
};

DataTable.Body = () => {
  const { sortedData } = useContext(DataTableContext);
  
  return (
    <tbody>
      {sortedData.map((row, index) => (
        <tr key={index}>
          {Object.values(row).map((cell, cellIndex) => (
            <td key={cellIndex}>{cell}</td>
          ))}
        </tr>
      ))}
    </tbody>
  );
};

// Usage
<DataTable data={users}>
  <DataTable.Header>
    <DataTable.Column field="name" sortable>Name</DataTable.Column>
    <DataTable.Column field="email">Email</DataTable.Column>
    <DataTable.Column field="role" sortable>Role</DataTable.Column>
  </DataTable.Header>
  <DataTable.Body />
</DataTable>
```

**Scalability Tip:** This pattern works best when the compound components share a clear, well-defined relationship. Avoid over-engineering—sometimes a simple prop-based API is more appropriate.

## Hooks-based Abstraction

Custom hooks provide a clean way to encapsulate stateful logic and side effects, making components more focused and reusable.

### Beyond Basic Examples

Here's a pattern I've used for handling complex form validation with async rules:

```tsx
const useFormValidation = (initialValues, validationRules) => {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [isValidating, setIsValidating] = useState({});
  const validationTimeouts = useRef({});

  const validateField = useCallback(async (fieldName, value) => {
    const rule = validationRules[fieldName];
    if (!rule) return;

    setIsValidating(prev => ({ ...prev, [fieldName]: true }));
    
    // Clear existing timeout
    if (validationTimeouts.current[fieldName]) {
      clearTimeout(validationTimeouts.current[fieldName]);
    }

    // Debounce validation
    validationTimeouts.current[fieldName] = setTimeout(async () => {
      try {
        const error = await rule(value, values);
        setErrors(prev => ({ ...prev, [fieldName]: error }));
      } catch (err) {
        setErrors(prev => ({ ...prev, [fieldName]: 'Validation failed' }));
      } finally {
        setIsValidating(prev => ({ ...prev, [fieldName]: false }));
      }
    }, 300);
  }, [validationRules, values]);

  const handleChange = useCallback((fieldName, value) => {
    setValues(prev => ({ ...prev, [fieldName]: value }));
    validateField(fieldName, value);
  }, [validateField]);

  const validateAll = useCallback(async () => {
    const validationPromises = Object.keys(validationRules).map(fieldName =>
      validateField(fieldName, values[fieldName])
    );
    
    await Promise.all(validationPromises);
    return Object.keys(errors).length === 0;
  }, [validationRules, values, errors, validateField]);

  return {
    values,
    errors,
    isValidating,
    handleChange,
    validateAll,
    setValues
  };
};

// Usage
const validationRules = {
  email: async (value) => {
    if (!value) return 'Email is required';
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) return 'Invalid email format';
    
    // Async validation
    const exists = await api.checkEmailExists(value);
    if (exists) return 'Email already exists';
    
    return null;
  },
  password: (value) => {
    if (!value) return 'Password is required';
    if (value.length < 8) return 'Password must be at least 8 characters';
    return null;
  }
};

const RegistrationForm = () => {
  const { values, errors, isValidating, handleChange, validateAll } = useFormValidation(
    { email: '', password: '' },
    validationRules
  );

  const handleSubmit = async (e) => {
    e.preventDefault();
    const isValid = await validateAll();
    if (isValid) {
      // Submit form
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={values.email}
        onChange={(e) => handleChange('email', e.target.value)}
        className={errors.email ? 'error' : ''}
      />
      {isValidating.email && <span>Validating...</span>}
      {errors.email && <span className="error">{errors.email}</span>}
      
      <input
        type="password"
        value={values.password}
        onChange={(e) => handleChange('password', e.target.value)}
        className={errors.password ? 'error' : ''}
      />
      {errors.password && <span className="error">{errors.password}</span>}
      
      <button type="submit">Register</button>
    </form>
  );
};
```

**Performance Consideration:** Be mindful of hook dependencies and avoid creating new objects/functions on every render. Use `useCallback` and `useMemo` strategically.

## Render Props Pattern vs Custom Hooks

Both patterns solve similar problems but have different trade-offs. Render props provide more flexibility, while custom hooks offer better performance and cleaner syntax.

### Render Props Example

```tsx
const DataFetcher = ({ url, children }) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url);
        const result = await response.json();
        setData(result);
      } catch (err) {
        setError(err);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [url]);

  return children({ data, loading, error });
};

// Usage
<DataFetcher url="/api/users">
  {({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <ErrorMessage error={error} />;
    return <UserList users={data} />;
  }}
</DataFetcher>
```

### Custom Hook Equivalent

```tsx
const useDataFetcher = (url) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url);
        const result = await response.json();
        setData(result);
      } catch (err) {
        setError(err);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [url]);

  return { data, loading, error };
};

// Usage
const UserListContainer = () => {
  const { data, loading, error } = useDataFetcher('/api/users');
  
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <UserList users={data} />;
};
```

**When to Use Each:**
- **Render Props:** When you need to conditionally render different components or when the consumer needs to control the rendering logic
- **Custom Hooks:** When you want better performance (no wrapper components) and cleaner component code

## Controlled vs Uncontrolled Components

This pattern choice affects how you manage form state and can significantly impact performance and user experience.

### Controlled Components

```tsx
const ControlledForm = () => {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    preferences: []
  });

  const handleChange = (field, value) => {
    setFormData(prev => ({ ...prev, [field]: value }));
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log('Form data:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={formData.name}
        onChange={(e) => handleChange('name', e.target.value)}
        placeholder="Name"
      />
      <input
        type="email"
        value={formData.email}
        onChange={(e) => handleChange('email', e.target.value)}
        placeholder="Email"
      />
      <button type="submit">Submit</button>
    </form>
  );
};
```

### Uncontrolled Components

```tsx
const UncontrolledForm = () => {
  const formRef = useRef();

  const handleSubmit = (e) => {
    e.preventDefault();
    const formData = new FormData(formRef.current);
    const data = Object.fromEntries(formData);
    console.log('Form data:', data);
  };

  return (
    <form ref={formRef} onSubmit={handleSubmit}>
      <input
        type="text"
        name="name"
        placeholder="Name"
        defaultValue=""
      />
      <input
        type="email"
        name="email"
        placeholder="Email"
        defaultValue=""
      />
      <button type="submit">Submit</button>
    </form>
  );
};
```

**Performance Insight:** For large forms with many fields, uncontrolled components can be more performant since they don't trigger re-renders on every keystroke. However, controlled components provide better validation and real-time feedback capabilities.

## Conclusion

These patterns aren't silver bullets—they're tools that serve specific purposes. The key is understanding when and why to use each one. In my experience, the most successful React applications use a combination of these patterns, adapting them to the specific needs of the project.

The best approach is to start simple and refactor toward these patterns as your application grows. Premature optimization can lead to unnecessary complexity.

---

*Have you encountered any of these patterns in your projects? I'd love to hear about your experiences and any additional patterns that have worked well for you. Feel free to contribute to this knowledge base with your own insights!*
