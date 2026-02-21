# Web Components

ML2 uses native Web Components for complex UI elements. The primary component is the Bible verse selector used throughout the application.

## Overview

Web Components provide:
- **Encapsulation**: Self-contained functionality
- **Reusability**: Use anywhere in the application
- **Standards-Based**: Native browser APIs, no frameworks needed

## Bible Verse Selector

The `<bible-verse-selector>` component provides an interactive interface for selecting Bible passages.

### Usage

```html
<bible-verse-selector
    data-book="01"
    data-chapter="001"
    data-start="001001001"
    data-end="001001010">
</bible-verse-selector>
```

### Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `data-book` | Book code (01-66) | `01` (Genesis) |
| `data-chapter` | Chapter number | `001` |
| `data-start` | Start verse ID | `001001001` (Gen 1:1) |
| `data-end` | End verse ID | `001001010` (Gen 1:10) |

### JavaScript API

```javascript
const selector = document.querySelector('bible-verse-selector');

// Get current selection
const selection = selector.getSelection();
// Returns: { book: '01', chapter: '001', start: '001001001', end: '001001010' }

// Set selection
selector.setSelection({
    book: '43',
    chapter: '003',
    start: '043003016',
    end: '043003016'
});

// Listen for changes
selector.addEventListener('change', (e) => {
    console.log('Selection changed:', e.detail);
});
```

### Events

| Event | Description | Detail |
|-------|-------------|--------|
| `change` | Selection changed | `{ book, chapter, start, end }` |
| `ready` | Component initialized | - |

## Component Architecture

### Class Structure

```javascript
class BibleVerseSelector extends HTMLElement {
    constructor() {
        super();
        // Initialize state
        this._state = {
            currentBook: '01',
            currentChapter: '001',
            selectedStart: null,
            selectedEnd: null
        };
        
        // Create DOM elements
        this._rootEl = document.createElement('div');
        // ...
    }
    
    connectedCallback() {
        // Component added to DOM
        this._loadData();
        this._render();
        this._attachEvents();
    }
    
    disconnectedCallback() {
        // Cleanup event listeners
    }
    
    // Public API methods
    getSelection() { /* ... */ }
    setSelection(data) { /* ... */ }
}

customElements.define('bible-verse-selector', BibleVerseSelector);
```

### State Management

Components manage internal state:

```javascript
this._state = {
    currentBook: '01',      // Currently displayed book
    currentChapter: '001',  // Currently displayed chapter
    selectedStart: null,    // First selected verse
    selectedEnd: null       // Last selected verse
};

this._bookMenu = {
    open: false,
    view: 'books',          // 'books' or 'categories'
    testament: 'OT'         // 'OT' or 'NT'
};
```

### Data Loading

Components load data from JSON endpoints:

```javascript
async _loadData() {
    // Load Bible structure
    const bible = await fetch('/Assets/data/bible.json').then(r => r.json());
    this._bible = bible;
    
    // Build indexes for fast lookup
    this._buildIndexes();
}
```

## Creating a Web Component

### Basic Template

```javascript
// my-component.js

class MyComponent extends HTMLElement {
    constructor() {
        super();
        
        // Initialize state
        this._state = {
            value: null
        };
        
        // Create shadow DOM (optional)
        this.attachShadow({ mode: 'open' });
    }
    
    // Called when element is added to DOM
    connectedCallback() {
        this._render();
        this._attachEvents();
    }
    
    // Called when element is removed
    disconnectedCallback() {
        // Cleanup
    }
    
    // Called when attributes change
    static get observedAttributes() {
        return ['data-value'];
    }
    
    attributeChangedCallback(name, oldValue, newValue) {
        if (name === 'data-value') {
            this._state.value = newValue;
            this._render();
        }
    }
    
    // Render component
    _render() {
        this.shadowRoot.innerHTML = `
            <style>
                :host { display: block; }
                .container { padding: 1rem; }
            </style>
            <div class="container">
                <slot></slot>
            </div>
        `;
    }
    
    // Attach event listeners
    _attachEvents() {
        this.addEventListener('click', this._handleClick);
    }
    
    _handleClick = (e) => {
        // Dispatch custom event
        this.dispatchEvent(new CustomEvent('action', {
            detail: { value: this._state.value },
            bubbles: true
        }));
    }
    
    // Public API
    getValue() {
        return this._state.value;
    }
    
    setValue(value) {
        this._state.value = value;
        this.setAttribute('data-value', value);
    }
}

customElements.define('my-component', MyComponent);
```

### Using in Templates

```php
<!-- Include the component script -->
<script type="module" src="/Assets/js/my-component.js"></script>

<!-- Use the component -->
<my-component data-value="initial">
    <p>Slot content</p>
</my-component>
```

## Best Practices

### 1. Use Shadow DOM for Encapsulation

```javascript
constructor() {
    super();
    this.attachShadow({ mode: 'open' });
}
```

### 2. Observe Only Necessary Attributes

```javascript
static get observedAttributes() {
    return ['data-value']; // Only these trigger attributeChangedCallback
}
```

### 3. Clean Up Event Listeners

```javascript
disconnectedCallback() {
    this.removeEventListener('click', this._handleClick);
    document.removeEventListener('keydown', this._handleKeydown);
}
```

### 4. Dispatch Custom Events for Communication

```javascript
this.dispatchEvent(new CustomEvent('change', {
    detail: { value: this._state.value },
    bubbles: true,      // Event bubbles up DOM
    composed: true      // Event crosses shadow DOM boundary
}));
```

### 5. Provide Public API Methods

```javascript
// Getter
getValue() {
    return this._state.value;
}

// Setter
setValue(value) {
    this._state.value = value;
    this._render();
}
```

## Component Lifecycle

```
constructor()
    ↓
connectedCallback()
    ↓
attributeChangedCallback() [if attributes change]
    ↓
[User interactions]
    ↓
disconnectedCallback()
```

## Styling Components

### Shadow DOM Styles

```javascript
_render() {
    this.shadowRoot.innerHTML = `
        <style>
            :host {
                display: block;
                --primary-color: #0055ff;
            }
            
            .button {
                background: var(--primary-color);
                color: white;
            }
            
            /* Style slotted content */
            ::slotted(p) {
                margin: 0;
            }
        </style>
        <button class="button"><slot></slot></button>
    `;
}
```

### CSS Custom Properties

Expose styling hooks:

```css
/* Component defines default */
:host {
    --my-component-primary: #0055ff;
    --my-component-padding: 1rem;
}

/* Parent page can override */
my-component {
    --my-component-primary: #ff5500;
}
```

## Testing Components

```javascript
// Test helper
function createComponent(tag, attributes = {}) {
    const el = document.createElement(tag);
    Object.entries(attributes).forEach(([key, value]) => {
        el.setAttribute(key, value);
    });
    document.body.appendChild(el);
    return el;
}

// Test
describe('MyComponent', () => {
    it('should render with default value', () => {
        const el = createComponent('my-component');
        assert(el.getValue() === null);
        el.remove();
    });
    
    it('should update value', () => {
        const el = createComponent('my-component');
        el.setValue('test');
        assert(el.getValue() === 'test');
        el.remove();
    });
});
```

---

**Status:** Accurate and Complete  
**Last Updated:** 2025-02-08
