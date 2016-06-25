+++
bigimg = ""
date = "2016-06-25T00:00:00+02:00"
subtitle = "How to make the user do what you need him to do"
title = "Cancel That Please"
tags = ["C#", "WPF", "TabControl", "Selector", "Cancel" ]
+++

I was working with the [TabControl](https://msdn.microsoft.com/en-us/library/system.windows.controls.tabcontrol(v=vs.110).aspx) the other day
 and I wanted to prevent the user from abandoning the task he was currently performing by cancelling the selection of a new tab. 

This was more dificult than expected because [Selector](https://msdn.microsoft.com/en-us/library/system.windows.controls.primitives.selector(v=vs.110).aspx)
 does not implement a SelectionChanging event with which you can cancel the selection changes.

Enter the  *SelectorAttachedProperties.HasActivatableSupport* dependency property. Together with an *IActivatable* interface that is implemented by the model for the tab page it will take care of the nasty details of cancelling a selection.

the *Activate* and *Deactivate* methods wil get called on the model when a tab is switched, and if  *Deactivate* returns false the tab won't switch.


[See the full code here](https://github.com/resc/wpfmagic/tree/master/SelectorAttachedPropertiesHasActivatableSupport)

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

References That Inspired This Post
====
1. [Stack Overflow](http://stackoverflow.com/questions/30706758/how-to-cancel-tab-change-in-wpf-tabcontrol)
2. [CodeRelief.NET](https://coderelief.net/2011/11/07/fixing-issynchronizedwithcurrentitem-and-icollectionview-cancel-bug-with-an-attached-property/)
