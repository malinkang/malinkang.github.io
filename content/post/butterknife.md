---
title: ButterKnife源码分析
date: 2015-04-19 10:40:23
draft: true
---

### 功能介绍
在Android中，我们从布局文件中获取一个view一般通过findViewById方法来操作，当一个界面里view过多的时候，我们需要花费大量的时间来写这些样板代码。ButterKnife用于来简化这个操作的，只需要在相应的View上加一个注解，框架就会注入这些字段。

<!--more-->

### 基本使用
在Activity中使用
```java
class ExampleActivity extends Activity {
  @Bind(R.id.title) TextView title;
  @Bind(R.id.subtitle) TextView subtitle;
  @Bind(R.id.footer) TextView footer;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}
```
在Fragment中使用
```java
public class FancyFragment extends Fragment {
  @Bind(R.id.button1) Button button1;
  @Bind(R.id.button2) Button button2;

  @Override public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    View view = inflater.inflate(R.layout.fancy_fragment, container, false);
    ButterKnife.bind(this, view);
    // TODO Use fields...
    return view;
  }
}
```
Adapter中使用
```java
public class MyAdapter extends BaseAdapter {
  @Override public View getView(int position, View view, ViewGroup parent) {
    ViewHolder holder;
    if (view != null) {
      holder = (ViewHolder) view.getTag();
    } else {
      view = inflater.inflate(R.layout.whatever, parent, false);
      holder = new ViewHolder(view);
      view.setTag(holder);
    }

    holder.name.setText("John Doe");
    // etc...

    return view;
  }

  static class ViewHolder {
    @Bind(R.id.title) TextView name;
    @Bind(R.id.job_title) TextView jobTitle;

    public ViewHolder(View view) {
      ButterKnife.bind(this, view);
    }
  }
}
```
多个View具有相同的操作我们可以使用@Bind注解一个View集合。
```java
@Bind({ R.id.first_name, R.id.middle_name, R.id.last_name })
List<EditText> nameViews;
```
Action和Setter接口用来设置操作
```java
static final Action<View> DISABLE = new Action<>() {
  @Override public void apply(View view, int index) {
    view.setEnabled(false);
  }
}

static final Setter<View, Boolean> ENABLED = new Setter<>() {
  @Override public void set(View view, Boolean value, int index) {
    view.setEnabled(value);
  }
}
```
调用apply方法传入view集合和操作。
```java
ButterKnife.apply(nameViews, DISABLE);
ButterKnife.apply(nameViews, ENABLED, false);
```

### 基本原理

ButterKnife在编译时刻利用APT将为所有使用ButterKnife的注解的类生成一个辅助的类，这个辅助类实现ViewBinder接口的bind方法，所有findViewById的操作都在bind方法中完成，在类中调用ButterKnife的bind方法其实是调用辅助类的bind方法。BtterKnife实现依赖注入的开销仅仅是在编译时刻做的注解处理，程序运行时的开销几乎可以忽略不计。
例如我们在SimpleActivity中使用ButterKnife

```java
public class SimpleActivity extends Activity {
  @Bind(R.id.title) TextView title;
  @Unbinder ViewUnbinder unbinder;
  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    title.setText("Butter Knife");
  }
}
```
ButterKnife为我们生成了辅助类SimpleActivity$$ViewBinder
```java
public class SimpleActivity$$ViewBinder<T extends SimpleActivity> implements ViewBinder<T> {
  @Override
  public void bind(final Finder finder, final T target, Object source) {
    Unbinder unbinder = createUnbinder(target);
    View view;
    view = finder.findRequiredView(source, 2130968576, "field 'title'");
    target.title = finder.castView(view, 2130968576, "field 'title'");
    target.unbinder = unbinder;
  }

  @SuppressWarnings("unchecked")
  protected <U extends Unbinder<T>> U createUnbinder(T target) {
    return (U) new Unbinder(target);
  }

  @SuppressWarnings("unchecked")
  protected <U extends Unbinder<T>> U accessUnbinder(T target) {
    return (U) target.unbinder;
  }

  public static class Unbinder<T extends SimpleActivity> implements ButterKnife.ViewUnbinder<T> {
    private T target;

    protected Unbinder(T target) {
      this.target = target;
    }

    @Override
    public final void unbind() {
      if (target == null) throw new IllegalStateException("Bindings already cleared.");
      unbind(target);
      target = null;
    }

    protected void unbind(T target) {
      target.title = null;
      target.unbinder = null;
    }
  }
}
```
### 获取注解信息

了解了基本原理之后，下面就看下ButterKnife如何获取这些信息，然后生成相应的类。获取信息的操作由`ButterKnifeProcessor`类完成，获取完成将注解类的信息封装成BindingClass对象。
```java
    @Override
    public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
        //调用findAndParseTargets获取所有的BindingClass
        Map<TypeElement, BindingClass> targetClassMap = findAndParseTargets(env);
        //遍历
    for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
            TypeElement typeElement = entry.getKey();
            BindingClass bindingClass = entry.getValue();

            try {//生成代码
                bindingClass.brewJava().writeTo(filer);
            } catch (IOException e) {
                error(typeElement, "Unable to write view binder for type %s: %s", typeElement,
                        e.getMessage());
            }
        }

        return true;
    }
```

```java
 private Map<TypeElement, BindingClass> findAndParseTargets(RoundEnvironment env) {
        Map<TypeElement, BindingClass> targetClassMap = new LinkedHashMap<>();
        Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();
        // Process each @Bind element.
        for (Element element : env.getElementsAnnotatedWith(Bind.class)) {
            //验证element
            if (!SuperficialValidation.validateElement(element)) continue;
            try {
                parseBind(element, targetClassMap, erasedTargetNames);
            } catch (Exception e) {
                logParsingError(element, Bind.class, e);
            }
        }

        // Process each annotation that corresponds to a listener.
        for (Class<? extends Annotation> listener : LISTENERS) {
            findAndParseListener(env, listener, targetClassMap, erasedTargetNames);
        }

        //...
        // Try to find a parent binder for each.
        for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
            TypeElement parentType = findParentType(entry.getKey(), erasedTargetNames);
            if (parentType != null) {
                String parentClassFqcn = getFqcn(parentType);
                BindingClass bindingClass = entry.getValue();
                bindingClass.setParentViewBinder(parentClassFqcn + BINDING_CLASS_SUFFIX);
                // Check if parent requested an unbinder.
                //判断父类使用使用@Bind进行注解
                BindingClass parentBindingClass = targetClassMap.get(parentType);
                if (parentBindingClass.hasUnbinder()) {
                    // Even if the child doesn't request an unbinder explicitly, we need to generate one.
                    if (!bindingClass.hasUnbinder()) {
                        bindingClass.requiresUnbinder(null);
                    }
                    // Check if the parent has a parent unbinder.
                    if (parentBindingClass.getParentUnbinder() != null) {
                        bindingClass.setParentUnbinder(parentBindingClass.getParentUnbinder());
                    } else {
                        bindingClass.setParentUnbinder(parentClassFqcn + BINDING_CLASS_SUFFIX + "."
                                + UnbinderBinding.UNBINDER_SIMPLE_NAME);
                    }
                }
            }
        }

        return targetClassMap;
    }

```
parseBind方法用来解析被@Bind注解的类。

```java
    private void parseBind(Element element, Map<TypeElement, BindingClass> targetClassMap,
                           Set<TypeElement> erasedTargetNames) {
        // Verify common generated code restrictions.
        if (isInaccessibleViaGeneratedCode(Bind.class, "fields", element)
                || isBindingInWrongPackage(Bind.class, element)) {
            return;
        }
        TypeMirror elementType = element.asType();

        if (elementType.getKind() == TypeKind.ARRAY) {        //类型是集合
            parseBindMany(element, targetClassMap, erasedTargetNames);
        } else if (LIST_TYPE.equals(doubleErasure(elementType))) { //类型是数组
            parseBindMany(element, targetClassMap, erasedTargetNames);
        } else if (isSubtypeOfType(elementType, ITERABLE_TYPE)) {// 其他可迭代的集合
            error(element, "@%s must be a List or array. (%s.%s)", Bind.class.getSimpleName(),
                    ((TypeElement) element.getEnclosingElement()).getQualifiedName(),
                    element.getSimpleName());
        } else {
            //error(element,"parse one");
            parseBindOne(element, targetClassMap, erasedTargetNames); //解析单个
        }
    }

```
parseBindMany用来解析被@Bind注解的数组或者集合
```java
    private void parseBindMany(Element element, Map<TypeElement, BindingClass> targetClassMap, Set<TypeElement> erasedTargetNames) {
        boolean hasError = false;
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();        //获取 enclosingElement
        // Verify that the type is a List or an array.
        TypeMirror elementType = element.asType();
        String erasedType = doubleErasure(elementType);//擦除被@Bind注解的集合的泛型
        TypeMirror viewType = null;//
        FieldCollectionViewBinding.Kind kind;
        if (elementType.getKind() == TypeKind.ARRAY) {
            ArrayType arrayType = (ArrayType) elementType;//强转为数组类型
            viewType = arrayType.getComponentType(); //获取数组元素类型
            kind = FieldCollectionViewBinding.Kind.ARRAY;
        } else if (LIST_TYPE.equals(erasedType)) {
            DeclaredType declaredType = (DeclaredType) elementType;
            //获取真实的类型 declaredType:List<View> -> 真实类型List
            List<? extends TypeMirror> typeArguments = declaredType.getTypeArguments();
            if (typeArguments.size() != 1) {
                error(element, "@%s List must have a generic component. (%s.%s)",
                        Bind.class.getSimpleName(), enclosingElement.getQualifiedName(),
                        element.getSimpleName());
                hasError = true;
            } else {
                viewType = typeArguments.get(0);
            }
            kind = FieldCollectionViewBinding.Kind.LIST;
        } else {
            throw new AssertionError();
        }
        if (viewType != null && viewType.getKind() == TypeKind.TYPEVAR) {
            TypeVariable typeVariable = (TypeVariable) viewType;
            viewType = typeVariable.getUpperBound(); //
        }

        // Verify that the target type extends from View.
        if (viewType != null && !isSubtypeOfType(viewType, VIEW_TYPE) && !isInterface(viewType)) {
            error(element, "@%s List or array type must extend from View or be an interface. (%s.%s)",
                    Bind.class.getSimpleName(), enclosingElement.getQualifiedName(), element.getSimpleName());
            hasError = true;
        }

        if (hasError) {
            return;
        }

        // Assemble information on the field.
        String name = element.getSimpleName().toString();
        int[] ids = element.getAnnotation(Bind.class).value();
        if (ids.length == 0) {
            error(element, "@%s must specify at least one ID. (%s.%s)", Bind.class.getSimpleName(),
                    enclosingElement.getQualifiedName(), element.getSimpleName());
            return;
        }
        //获取重复的id
        Integer duplicateId = findDuplicate(ids);
        if (duplicateId != null) {
            error(element, "@%s annotation contains duplicate ID %d. (%s.%s)", Bind.class.getSimpleName(),
                    duplicateId, enclosingElement.getQualifiedName(), element.getSimpleName());
        }

        assert viewType != null; // Always false as hasError would have been true.
        TypeName type = TypeName.get(viewType);
        boolean required = isFieldRequired(element);
        //创建BindingClass类
        BindingClass bindingClass = getOrCreateTargetClass(targetClassMap, enclosingElement);
        //FieldCollectionViewBinding封装了@Bind注解的集合或数组的信息
        FieldCollectionViewBinding binding = new FieldCollectionViewBinding(name, type, kind, required);
        bindingClass.addFieldCollection(ids, binding);

        erasedTargetNames.add(enclosingElement);
    }
```
其他的parse操作大同小异，所以这里就不给出代码分析。

### 生成代码
BindingClass的brewJava()方法构建一个JavaFile对象。
```java
  JavaFile brewJava() {
    //添加泛型<T extends targetClass>
    TypeSpec.Builder result = TypeSpec.classBuilder(className)
        .addModifiers(PUBLIC)
        .addTypeVariable(TypeVariableName.get("T", ClassName.bestGuess(targetClass)));
    //
    if (parentViewBinder != null) {
      result.superclass(ParameterizedTypeName.get(ClassName.bestGuess(parentViewBinder),
          TypeVariableName.get("T")));
    } else {
      result.addSuperinterface(ParameterizedTypeName.get(VIEW_BINDER, TypeVariableName.get("T")));
    }

    result.addMethod(createBindMethod());

    if (hasUnbinder()) {
      // Create unbinding class.
      result.addType(createUnbinderClass());
      // Now we need to provide child classes to access and override unbinder implementations.
      createUnbinderInternalAccessMethods(result);
    }

    return JavaFile.builder(classPackage, result.build())
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
  }

```

### ButterKnife的bind方法

```java
  static void bind(@NonNull Object target, @NonNull Object source, @NonNull Finder finder) {
    Class<?> targetClass = target.getClass();
    try {
      if (debug) Log.d(TAG, "Looking up view binder for " + targetClass.getName());
      //获取生成的辅助类
      ViewBinder<Object> viewBinder = findViewBinderForClass(targetClass);
      //获取辅助类的bind方法
      viewBinder.bind(finder, target, source);
    } catch (Exception e) {
      throw new RuntimeException("Unable to bind views for " + targetClass.getName(), e);
    }
  }

    @NonNull
  private static ViewBinder<Object> findViewBinderForClass(Class<?> cls)
      throws IllegalAccessException, InstantiationException {
    ViewBinder<Object> viewBinder = BINDERS.get(cls);
    if (viewBinder != null) {
      if (debug) Log.d(TAG, "HIT: Cached in view binder map.");
      return viewBinder;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return NOP_VIEW_BINDER;
    }
    try {
      Class<?> viewBindingClass = Class.forName(clsName + "$$ViewBinder");
      //noinspection unchecked
      viewBinder = (ViewBinder<Object>) viewBindingClass.newInstance();
      if (debug) Log.d(TAG, "HIT: Loaded view binder class.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      viewBinder = findViewBinderForClass(cls.getSuperclass());
    }
    BINDERS.put(cls, viewBinder);
    return viewBinder;
  }


```
