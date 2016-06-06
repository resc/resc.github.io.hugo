+++
bigimg = ""
date = "2016-06-05T11:05:34+02:00"
subtitle = "How to make the user do what you need him to do"
title = "Cancel That Please"
tags = ["C#", "WPF", "TabControl", "Selector" ]
+++

I was working with the [TabControl](https://msdn.microsoft.com/en-us/library/system.windows.controls.tabcontrol(v=vs.110).aspx) the other day and I wanted to prevent the user from abandoning the task he was currently performing by cancelling the selection of a new tab. This was more dificult than expected because controls derived from Selector do not implement a SelectionChanging event with which you can cancel the selection change.



So, you've got users that are motivationally impaired, cognitively challenged, or they just cannot avoid that ID-10-T error code. And you want to help them get though life with a minimum of frustration by implementing a "Are You Sure?" ( which is annoying to them and makes them doubt themselves, *mwuhahahaha* ) or an auto-save feature ( You're such a nice person, I can tell! ).

Enter the  SelectorAttachedProperties.HasActivatableSupport dependency property.


```xaml
 <TabControl utils:SelectorAttachedProperties.HasActivatableSupport="True">
  <!-- ... -->
</TabControl>
```



```csharp
    /// <summary> IActivatable should be implemented by a ViewModel. </summary>
    public interface IActivatable
    {
        /// <summary> Called when the module is activated. </summary>
        void Activate();

        /// <summary> Return true to continue with deactivation, false to cancel deactivation. </summary>
        bool Deactivate();
    }

    public static class SelectorAttachedProperties
    {
        private static readonly Type _ownerType = typeof(SelectorAttachedProperties);

        private static readonly ConditionalWeakTable<Selector, ActivationHandler> _handlers = new ConditionalWeakTable<Selector, ActivationHandler>();


        public static readonly DependencyProperty HasActivatableSupportProperty =
            DependencyProperty.RegisterAttached("HasActivatableSupport", typeof(bool), _ownerType,
            new PropertyMetadata(false, HasActivatableSupportChanged));

        public static bool GetHasActivatableSupport(DependencyObject obj)
        {
            return (bool)obj.GetValue(HasActivatableSupportProperty);
        }

        public static void SetHasActivatableSupport(DependencyObject obj, bool value)
        {
            obj.SetValue(HasActivatableSupportProperty, value);
        }

        private static void HasActivatableSupportChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            var selector = d as Selector;
            if (selector == null || !(e.OldValue is bool && e.NewValue is bool) || e.OldValue == e.NewValue)
                return;

            var enableActivatableSupport = (bool)e.NewValue;
            var handler = _handlers.GetOrCreateValue(selector);

            TypeDescriptor.GetProperties(selector)["ItemsSource"].RemoveValueChanged(selector, handler.OnItemsSourceChanged);
            selector.SelectionChanged -= handler.OnSelectionChanged;

            if (enableActivatableSupport)
            {
                selector.IsSynchronizedWithCurrentItem = true;
                TypeDescriptor.GetProperties(selector)["ItemsSource"].AddValueChanged(selector, handler.OnItemsSourceChanged);
                selector.SelectionChanged += handler.OnSelectionChanged;
            }
        }

        private class ActivationHandler
        {
            ICollectionView _collectionView;

            public void OnItemsSourceChanged(object sender, EventArgs e)
            {
                var selector = sender as Selector;
                if (selector != null)
                {
                    var itemsSource = selector.ItemsSource;
                    _collectionView = itemsSource as ICollectionView ?? CollectionViewSource.GetDefaultView(itemsSource);
                    _collectionView.CurrentChanging += OnCurrentChanging;
                    _collectionView.CurrentChanged += OnCurrentChanged;
                }
            }

            public void OnSelectionChanged(object sender, SelectionChangedEventArgs args)
            {
                var collectionView = _collectionView;
                if (collectionView == null) return;

                var selector = sender as Selector;
                if (selector == null) return;

                if (selector.IsSynchronizedWithCurrentItem == true && selector.SelectedItem != collectionView.CurrentItem)
                {
                    selector.IsSynchronizedWithCurrentItem = false;
                    selector.SelectedItem = collectionView.CurrentItem;
                    selector.IsSynchronizedWithCurrentItem = true;
                }
            }

            private void OnCurrentChanging(object sender, CurrentChangingEventArgs e)
            {
                var activatable = (sender as ICollectionView)?.CurrentItem as IActivatable;
                if (activatable?.Deactivate() == false)
                {
                    if (e.IsCancelable)
                        e.Cancel = true;
                }
            }

            private void OnCurrentChanged(object sender, EventArgs e)
            {
                var activatable = (sender as ICollectionView)?.CurrentItem as IActivatable;
                activatable?.Activate();
            }
        }
    }

```

References
====
1. [Stack Overflow](http://stackoverflow.com/questions/30706758/how-to-cancel-tab-change-in-wpf-tabcontrol)
2. [CodeRelief.NET](https://coderelief.net/2011/11/07/fixing-issynchronizedwithcurrentitem-and-icollectionview-cancel-bug-with-an-attached-property/)
