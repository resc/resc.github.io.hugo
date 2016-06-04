+++
date = "2016-06-04T11:10:02+02:00"
draft = true
title = "WPF Magic Without Frameworks"
subtitle = "How to build simple user interfaces without big frameworks"
tags = [ "Development", "C#", "WPF" ]
+++

Building WPF applications is challenging is you're new to WPF.
There are lots of frameworks, components and toolkits to choose from and WPF is no small framework itself.
I'll show you some tricks to build a WPF UI without toolkits but with magic.

But first some basic concepts I'll use a lot.

1. A *View* is WPF FrameworkElement or a control derived from it. 
2. A  *ViewModel* will be (in) the [DataContext](https://msdn.microsoft.com/en-us/library/system.windows.frameworkelement.datacontext(v=vs.110).aspx) of the view and contain everything the view needs to do its viewy things.
3. A *DataModel* contains the actual data that the user cares about and will be put in properties on the viewmodel.
4. *[DataTemplates](https://msdn.microsoft.com/en-us/library/ms742521(v=vs.100).aspx)* are WPF's way to say "I want this model bound to this view structure"
5. *[Styles](https://msdn.microsoft.com/en-us/library/ms742521(v=vs.100).aspx)* are WPF's way to say "I want this view to be configured in this way"

Show Me The Magic!
====

Ok, I'm going to make Views magically appear when we bind a ContentControl.Content property to a model.
and because a good programmer is a lazy programmer I'll use convention over configuration to do so.

The conventions are:

* View type names end with 'View'
* Model types names end with 'Model'
* View and Model live right next to each other in the same namespace and assembly.
* ContentControls are magic.

To turn the magic on I'll create a custom DataTemplateSelector that finds the right View for the model bound to the ContentControl.Content property. and set the ContentControl.ContentTemplateSector with it.

And because magic is only magic if it happens without manual labour I'll put a Style for ContentControl in the application level resources so every ContentControl gets that ContentTemplateSelector. 

In App.xaml
```xaml
<Application.Resources>
    <!--Create the DataTemplateSelector -->
    <conventions:ConventionDataTemplateSelector x:Key="ConventionDataTemplateSelector" />
    
    <!-- Modify ContentControl to use ConventionDataTemplateSelector for its Content-->
    <Style TargetType="ContentControl">
        <Setter Property="ContentTemplateSelector" Value="{StaticResource ConventionDataTemplateSelector}" />
    </Style>
</Application.Resources>
```

And the code for the ConventionDataTemplateSelector:

```csharp
   
    public class ConventionDataTemplateSelector : DataTemplateSelector
    {
        // cache the templates until the model goes away
        private readonly ConditionalWeakTable<object, DataTemplate> _templatesCache = new ConditionalWeakTable<object, DataTemplate>();

        const string Model = "Model";
        const string View = "View";

        public override DataTemplate SelectTemplate(object item, DependencyObject container)
        {
            // Don't use the selector in design mode in Visual Studio
            if (DesignerProperties.GetIsInDesignMode(container))
                return base.SelectTemplate(item, container);

            if (item != null)
            {
                lock (_templatesCache)
                {
                    DataTemplate template;
                    if (_templatesCache.TryGetValue(item, out template))
                        return template;

                    var templateType = GetTemplateTypeFor(item, container);
                    template = new DataTemplate
                    {
                        VisualTree = new FrameworkElementFactory
                        {
                            Type = templateType
                        }
                    };

                    _templatesCache.Add(item, template);
                    return template;
                }
            }

            return base.SelectTemplate(null, container);
        }

        public virtual Type GetTemplateTypeFor(object item, DependencyObject container)
        {
            var type = item.GetType();
            try
            {
                if (!type.Name.EndsWith(Model, StringComparison.Ordinal))
                {
                    throw new TypeLoadException($"type {type} does not conform to the conventions or a viewmodel," +
                                                $" the type's name should end with '{Model}'");
                }

                var viewTypeName = type.FullName;
                viewTypeName = viewTypeName.Substring(0, viewTypeName.Length - Model.Length) + View;

                try
                {
                    // Load the view type from the same assembly as the model type.
                    var templateTypeFor = type.Assembly.GetType(viewTypeName, true);
                    return templateTypeFor;
                }
                catch (Exception e)
                {
                    throw new TypeLoadException($"{GetType().Name}: Error loading view type {viewTypeName} for model {type}: {e.Message}", e);
                }
            }
            catch (TypeLoadException)
            {
                throw;
            }
            catch (Exception e)
            {
                throw new TypeLoadException($"{GetType().Name}: Error loading view for model {type}: {e.Message}", e);
            }
        }
    }

```


See [the full code here](https://github.com/resc/wpfmagic/tree/master/ConventionDataTemplateSelector).



