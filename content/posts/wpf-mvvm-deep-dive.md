---
title: "WPF MVVM模式深度解析：原理与实践"
date: 2024-03-18
draft: false
tags: ["WPF", "MVVM", ".NET", "桌面开发", "设计模式"]
categories: ["桌面应用开发"]
---

# WPF MVVM模式深度解析：原理与实践

MVVM（Model-View-ViewModel）是WPF应用程序开发中最重要的设计模式之一。本文将深入探讨MVVM的核心概念、实现原理以及最佳实践，帮助您构建可维护、可测试的WPF应用程序。

## MVVM基础架构

### 基础类实现

```csharp
public abstract class ViewModelBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;

    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }

    protected bool SetProperty<T>(ref T field, T value, [CallerMemberName] string propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value)) return false;
        field = value;
        OnPropertyChanged(propertyName);
        return true;
    }
}
```

### 命令实现

```csharp
public class RelayCommand : ICommand
{
    private readonly Action<object> _execute;
    private readonly Func<object, bool> _canExecute;

    public event EventHandler CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }

    public RelayCommand(Action<object> execute, Func<object, bool> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter)
    {
        return _canExecute == null || _canExecute(parameter);
    }

    public void Execute(object parameter)
    {
        _execute(parameter);
    }
}
```

## 数据绑定

### 属性绑定

```csharp
public class UserViewModel : ViewModelBase
{
    private string _username;
    private string _email;
    private bool _isActive;

    public string Username
    {
        get => _username;
        set => SetProperty(ref _username, value);
    }

    public string Email
    {
        get => _email;
        set
        {
            if (SetProperty(ref _email, value))
            {
                ValidateEmail();
            }
        }
    }

    public bool IsActive
    {
        get => _isActive;
        set
        {
            if (SetProperty(ref _isActive, value))
            {
                OnActiveStatusChanged();
            }
        }
    }

    private void ValidateEmail()
    {
        // 邮箱验证逻辑
    }

    private void OnActiveStatusChanged()
    {
        // 状态变更处理
    }
}
```

### 集合绑定

```csharp
public class ProductsViewModel : ViewModelBase
{
    private readonly ObservableCollection<ProductViewModel> _products;
    private ProductViewModel _selectedProduct;

    public ProductsViewModel()
    {
        _products = new ObservableCollection<ProductViewModel>();
        LoadProducts();
    }

    public ObservableCollection<ProductViewModel> Products => _products;

    public ProductViewModel SelectedProduct
    {
        get => _selectedProduct;
        set
        {
            if (SetProperty(ref _selectedProduct, value))
            {
                OnSelectedProductChanged();
            }
        }
    }

    private async void LoadProducts()
    {
        var products = await _productService.GetProductsAsync();
        foreach (var product in products)
        {
            _products.Add(new ProductViewModel(product));
        }
    }
}
```

## 依赖注入

### 服务注册

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // 注册服务
        services.AddSingleton<IProductService, ProductService>();
        services.AddSingleton<IUserService, UserService>();
        
        // 注册ViewModels
        services.AddTransient<MainViewModel>();
        services.AddTransient<ProductsViewModel>();
        services.AddTransient<UserViewModel>();
    }
}
```

### ViewModel注入

```csharp
public class MainViewModel : ViewModelBase
{
    private readonly IProductService _productService;
    private readonly IUserService _userService;

    public MainViewModel(
        IProductService productService,
        IUserService userService)
    {
        _productService = productService;
        _userService = userService;
        
        InitializeCommands();
    }

    private void InitializeCommands()
    {
        SaveCommand = new RelayCommand(ExecuteSave, CanExecuteSave);
        RefreshCommand = new RelayCommand(ExecuteRefresh);
    }
}
```

## 导航服务

### 导航实现

```csharp
public interface INavigationService
{
    void NavigateTo<T>() where T : ViewModelBase;
    void NavigateBack();
}

public class NavigationService : INavigationService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly Stack<ViewModelBase> _navigationStack;
    
    public event EventHandler<ViewModelBase> CurrentViewModelChanged;

    public NavigationService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
        _navigationStack = new Stack<ViewModelBase>();
    }

    public void NavigateTo<T>() where T : ViewModelBase
    {
        var viewModel = _serviceProvider.GetRequiredService<T>();
        _navigationStack.Push(viewModel);
        CurrentViewModelChanged?.Invoke(this, viewModel);
    }

    public void NavigateBack()
    {
        if (_navigationStack.Count > 1)
        {
            _navigationStack.Pop();
            var previousViewModel = _navigationStack.Peek();
            CurrentViewModelChanged?.Invoke(this, previousViewModel);
        }
    }
}
```

## 消息传递

### 消息中心

```csharp
public class MessageBus
{
    private static readonly MessageBus Instance = new();
    private readonly Dictionary<Type, List<object>> _subscribers;

    private MessageBus()
    {
        _subscribers = new Dictionary<Type, List<object>>();
    }

    public static MessageBus Default => Instance;

    public void Subscribe<TMessage>(Action<TMessage> action)
    {
        var type = typeof(TMessage);
        if (!_subscribers.ContainsKey(type))
        {
            _subscribers[type] = new List<object>();
        }
        _subscribers[type].Add(action);
    }

    public void Publish<TMessage>(TMessage message)
    {
        var type = typeof(TMessage);
        if (_subscribers.ContainsKey(type))
        {
            foreach (var subscriber in _subscribers[type].OfType<Action<TMessage>>())
            {
                subscriber(message);
            }
        }
    }
}
```

### 消息使用

```csharp
public class ProductUpdatedMessage
{
    public int ProductId { get; set; }
    public string ProductName { get; set; }
}

public class ProductViewModel : ViewModelBase
{
    public ProductViewModel()
    {
        MessageBus.Default.Subscribe<ProductUpdatedMessage>(OnProductUpdated);
    }

    private void OnProductUpdated(ProductUpdatedMessage message)
    {
        if (ProductId == message.ProductId)
        {
            ProductName = message.ProductName;
        }
    }
}
```

## 验证

### 数据验证

```csharp
public class UserViewModel : ViewModelBase, INotifyDataErrorInfo
{
    private readonly Dictionary<string, List<string>> _errors;
    
    public event EventHandler<DataErrorsChangedEventArgs> ErrorsChanged;

    public bool HasErrors => _errors.Any();

    public IEnumerable GetErrors(string propertyName)
    {
        return _errors.ContainsKey(propertyName) ? _errors[propertyName] : null;
    }

    protected void ValidateProperty(string propertyName)
    {
        var errors = new List<string>();
        
        switch (propertyName)
        {
            case nameof(Email):
                if (string.IsNullOrEmpty(Email))
                {
                    errors.Add("邮箱不能为空");
                }
                else if (!Regex.IsMatch(Email, @"^[^@\s]+@[^@\s]+\.[^@\s]+$"))
                {
                    errors.Add("邮箱格式不正确");
                }
                break;
        }

        if (errors.Any())
        {
            _errors[propertyName] = errors;
        }
        else
        {
            _errors.Remove(propertyName);
        }

        ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(propertyName));
    }
}
```

## 单元测试

### ViewModel测试

```csharp
[TestClass]
public class UserViewModelTests
{
    private Mock<IUserService> _userServiceMock;
    private UserViewModel _viewModel;

    [TestInitialize]
    public void Setup()
    {
        _userServiceMock = new Mock<IUserService>();
        _viewModel = new UserViewModel(_userServiceMock.Object);
    }

    [TestMethod]
    public void SaveCommand_WithValidData_CallsUserService()
    {
        // Arrange
        _viewModel.Username = "testuser";
        _viewModel.Email = "test@example.com";

        // Act
        _viewModel.SaveCommand.Execute(null);

        // Assert
        _userServiceMock.Verify(x => x.SaveUser(It.IsAny<UserModel>()), Times.Once);
    }

    [TestMethod]
    public void Email_WithInvalidFormat_HasValidationError()
    {
        // Arrange
        string invalidEmail = "invalid-email";

        // Act
        _viewModel.Email = invalidEmail;

        // Assert
        Assert.IsTrue(_viewModel.HasErrors);
        var errors = _viewModel.GetErrors(nameof(_viewModel.Email));
        Assert.IsNotNull(errors);
    }
}
```

## 性能优化

### 绑定优化

```csharp
public class LazyLoadingViewModel : ViewModelBase
{
    private readonly Lazy<ExpensiveData> _expensiveData;
    
    public LazyLoadingViewModel()
    {
        _expensiveData = new Lazy<ExpensiveData>(() =>
        {
            // 延迟加载昂贵的数据
            return LoadExpensiveData();
        });
    }

    public ExpensiveData Data => _expensiveData.Value;
}
```

### 虚拟化支持

```xaml
<ListView VirtualizingPanel.IsVirtualizing="True"
          VirtualizingPanel.VirtualizationMode="Recycling"
          ScrollViewer.IsDeferredScrollingEnabled="True"
          ItemsSource="{Binding LargeCollection}">
    <ListView.ItemTemplate>
        <DataTemplate>
            <StackPanel>
                <TextBlock Text="{Binding Name}" />
                <TextBlock Text="{Binding Description}" />
            </StackPanel>
        </DataTemplate>
    </ListView.ItemTemplate>
</ListView>
```

## 最佳实践

1. ViewModel设计原则
   - 保持ViewModel的独立性
   - 避免在ViewModel中直接操作View
   - 使用命令而不是事件
   - 实现适当的接口

2. 数据绑定建议
   - 使用适当的绑定模式
   - 注意集合更新的性能影响
   - 合理使用虚拟化
   - 处理大数据集时考虑分页

3. 代码组织
   - 遵循单一职责原则
   - 使用依赖注入
   - 实现适当的接口
   - 保持代码的可测试性

## 常见问题解决

1. 内存泄漏
```csharp
public class ViewModel : ViewModelBase, IDisposable
{
    private bool _disposed;
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // 清理托管资源
                UnsubscribeFromEvents();
            }
            
            _disposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

2. UI更新问题
```csharp
public class ViewModel : ViewModelBase
{
    private async Task UpdateUIAsync()
    {
        await Application.Current.Dispatcher.InvokeAsync(() =>
        {
            // UI更新代码
            UpdateUI();
        });
    }
}
```

## 总结

MVVM模式是WPF应用程序开发中的核心模式，通过本文我们详细探讨了：
- MVVM的基础架构
- 数据绑定机制
- 依赖注入的应用
- 导航和消息传递
- 验证和测试
- 性能优化策略

掌握这些知识和最佳实践，将帮助您开发出更加健壮、可维护的WPF应用程序。

## 参考资源

- WPF官方文档
- Microsoft MVVM指南
- .NET桌面应用开发最佳实践
- WPF性能优化指南
