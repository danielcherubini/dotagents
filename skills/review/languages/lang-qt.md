# Qt Code Review Guide

Qt-specific code review guide covering object model, signals/slots, memory management, thread safety, and performance. Covers Qt 6.x features.

## Table of Contents

- [Object Model](#object-model)
- [Signals & Slots](#signals--slots)
- [Memory Management](#memory-management)
- [Thread Safety](#thread-safety)
- [Performance](#performance)
- [Review Checklist](#qt-review-checklist)

---

## Object Model

### Parent-Child Ownership

```cpp
// ❌ Manual memory management with QObject children
QWidget *parent = new QWidget();
QPushButton *button = new QPushButton("Click", parent);  // OK: parent takes ownership
delete button;  // Double delete! parent will also delete it

// ✅ Let parent manage children
QWidget *parent = new QWidget();
new QPushButton("Click", parent);  // Parent deletes automatically

// ❌ Child without parent when it should have one
QLabel *label = new QLabel("Text");  // Memory leak if not added to widget tree
```

**Review checklist:**
- [ ] QObject children have appropriate parents
- [ ] No double deletion (child deleted by both code and parent)
- [ ] Raw pointers to QObjects used only for non-owning references

### Smart Pointers with QObject

```cpp
// ❌ unique_ptr with QObject (deletes via QObject destructor, not QObjectPrivate)
auto obj = std::make_unique<MyQObject>();

// ✅ Use qRegisterMetaType or avoid smart pointers for QObject-derived classes
// Qt's parent-child system is designed around raw pointers
MyQObject *obj = new MyQObject();
obj->setParent(someParent);
```

---

## Signals & Slots

### Connection Types

```cpp
// ❌ Auto connection in cross-thread scenarios (undefined behavior)
// Worker thread emits signal, GUI thread connects with Qt::AutoConnection
worker->signal.connect(guiObject, &GUIObject::slot);  // May crash!

// ✅ Explicitly specify connection type for cross-thread
worker->signal.connect(guiObject, &GUIObject::slot, Qt::QueuedConnection);

// ✅ Direct connection within same thread (more efficient)
connect(button, &QPushButton::clicked, this, &MyWidget::handleClick);
```

### Lambda Connections (Qt 5+)

```cpp
// ❌ Lambda capturing `this` can cause dangling pointers
connect(timer, &QTimer::timeout, this, [this]() {
    processData();  // If `this` is destroyed while timer runs: crash!
});

// ✅ Use QObject pointer capture (Qt 5.10+) or disconnect on destruction
connect(timer, &QTimer::timeout, this, [weak = QPointer(this)]() {
    if (weak) weak->processData();
});

// ✅ Or disconnect in destructor
~MyWidget() { timer->disconnect(); }
```

### Signal/Slot Best Practices

| Practice | Reason |
|----------|--------|
| Use function pointers over strings | Compile-time checking |
| Keep signals public, slots implementation detail | Encapsulation |
| Avoid signals in constructors/destructors | Object not fully initialized/destroyed |
| Disconnect when object destroyed | Prevent dangling connections |

---

## Memory Management

### QObject Lifecycle

```cpp
// ❌ Deleting parent doesn't delete children if parent has no parent
QWidget *widget = new QWidget();  // No parent, no automatic cleanup
delete widget;  // Children leak!

// ✅ Set up proper parent hierarchy
QWidget *parent = new QWidget();
new QPushButton("Click", parent);  // Parent owns child
// Deleting parent deletes all children

// ❌ Stale pointers after deletion
QPushButton *button = new QPushButton();
button->deleteLater();
button->setText("Error!");  // Use-after-delete!

// ✅ Use deleteLater() properly
button->deleteLater();
// Don't access button after this call
```

### Memory Leak Patterns

| Pattern | Fix |
|---------|-----|
| QObject without parent | Set appropriate parent or manage manually |
| Timer not stopped | Call `timer->stop()` in destructor |
| Event handlers not removed | Disconnect signals or remove event filters |
| Circular references with QPointer | Use QWeakPointer or break cycles |

---

## Thread Safety

### Threading Model

```cpp
// ❌ Accessing GUI from worker thread
void Worker::processData() {
    // This crashes: modifying QWidget from non-GUI thread
    label->setText("Done");
}

// ✅ Use signals/slots for cross-thread communication
connect(this, &Worker::progressChanged, this, &MyWidget::updateProgress);

// ✅ Or use Qt::QueuedConnection explicitly
connect(worker, &Worker::resultReady, this, &MyWidget::handleResult,
        Qt::QueuedConnection);
```

### QThread Best Practices

| Approach | When to Use |
|----------|-------------|
| `QThread` with slots | Simple background tasks |
| `QtConcurrent` | Parallel algorithms, map/reduce |
| `std::thread` + Qt integration | Complex threading needs |
| `QThreadPool` | Short-lived tasks |

```cpp
// ❌ Subclassing QThread and running long operations in run()
class Worker : public QThread {
    void run() override {
        // Long-running operation here
    }
};

// ✅ Preferred: Move worker object to thread
class WorkerObject : public QObject {
    Q_OBJECT
public slots:
    void doWork() { /* ... */ }
};

WorkerObject *worker = new WorkerObject();
QThread *thread = new QThread();
worker->moveToThread(thread);
connect(thread, &QThread::started, worker, &WorkerObject::doWork);
thread->start();
```

### Thread Safety Checklist

- [ ] GUI objects only accessed from main thread
- [ ] Cross-thread communication uses signals/slots with QueuedConnection
- [ ] Shared data protected by mutexes (QMutex/QReadWriteLock)
- [ ] No shared mutable state without synchronization
- [ ] Worker threads properly stopped on shutdown

---

## Performance

### Avoid Unnecessary Repaints

```cpp
// ❌ Forcing repaint in tight loop
for (int i = 0; i < 1000; i++) {
    widget->update();  // Triggers repaint each iteration
}

// ✅ Batch updates
widget->setUpdatesEnabled(false);
for (int i = 0; i < 1000; i++) {
    updateData(i);
}
widget->setUpdatesEnabled(true);
widget->update();  // Single repaint
```

### Model/View Optimization

| Technique | Benefit |
|-----------|---------|
| Lazy loading in models | Reduces memory for large datasets |
| `Qt::Quick` for complex UIs | GPU-accelerated rendering |
| Avoid `emit` in hot loops | Use direct function calls instead |
| Pre-compute frequently accessed data | Reduce repeated calculations |

### Container Performance

```cpp
// ❌ Using QList with many insertions at front
QList<int> list;
for (int i = 0; i < 10000; i++) {
    list.prepend(i);  // O(n) per insertion
}

// ✅ Use QQueue or QVector for append-heavy workloads
QVector<int> vec;
vec.reserve(10000);  // Pre-allocate
for (int i = 0; i < 10000; i++) {
    vec.append(i);  // O(1) amortized
}
```

---

## Qt Review Checklist

### Object Model
- [ ] QObject children have appropriate parents
- [ ] No double deletion of objects
- [ ] Raw pointers used correctly (owning vs non-owning)

### Signals & Slots
- [ ] Cross-thread connections use Qt::QueuedConnection
- [ ] Lambda captures safe (no dangling `this` pointers)
- [ ] Signals/slots properly disconnected on destruction

### Memory Management
- [ ] No memory leaks in object hierarchy
- [ ] Timers stopped before object destruction
- [ ] deleteLater() used for deferred deletion

### Thread Safety
- [ ] GUI only accessed from main thread
- [ ] Shared data protected by synchronization primitives
- [ ] Worker threads properly managed and stopped

### Performance
- [ ] Unnecessary repaints avoided (setUpdatesEnabled)
- [ ] Appropriate containers used (QVector vs QList vs QQueue)
- [ ] Model/View properly optimized for large datasets

### Code Quality
- [ ] Qt macros used correctly (Q_OBJECT, Q_GADGET)
- [ ] Constructor initialization lists used
- [ ] No raw new/delete where Qt's parent system suffices
- [ ] Error handling with appropriate return values

---

## Qt 6.10+ New Features

### Flexbox Layout for Qt Quick
```qml
// ✅ Qt 6.10+: Flexbox layout in Qt Quick
FlexRow {
    spacing: 8
    FlexItem { flexGrow: 1 }
    FlexItem { /* fixed */ }
}
```

### SVG and Lottie Animations
- Qt 6.10+ supports more vector animations in SVG and Lottie format
- Use `QSvgRenderer` for static SVG, `QAnimatedSvg` for animated

## Reference Resources

- [Qt Object Model](https://doc.qt.io/qt-6/object.html)
- [Signals and Slots](https://doc.qt.io/qt-6/signalsandslots.html)
- [Threading in Qt](https://doc.qt.io/qt-6/thread-basics.html)
- [Qt Performance Tips](https://doc.qt.io/qt-6/performance.html)
- [Qt 6.10 Release Notes](https://www.qt.io/blog/qt-6.10-released)
