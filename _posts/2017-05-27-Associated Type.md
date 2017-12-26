---
layout: post
comments: true
title:  "A thought about associated type"
date:   2017-05-27 15:32:24 +0700
categories: jekyll update
---

###Problem
Last week, I had to implement some table views which includes some custom cells. Those cells are totally different for each table. And I will need to continue on a lot of screens. I started thinking how to do it in a generic way, easy to implement for all screens and also customizable.


###Idea
I started with MVVM pattern. To recap, the main idea of MVVM is the view-model layer. It provides methods to map from model's data to ui's data and also handle incoming events from user interface. So that, the idea here is to create abstractions of the view-model class, and a general table view controller which will handle all view-models. Each time, we have to render a table view, we will only need to prepare its view model class and tada, everything should be set without any extra implementation of `UITableviewDataSource`. Sound good? Let see how it is in implementation.

###Impletation
* First I define 2 protocols for our datasources and custom cells

{% highlight swift %}
protocol BECellDataSource {
    func cellIdentifier() -> String
    static func register(tableView: UITableView)
}

protocol BECellRender {
    func renderCell(data: BECellDataSource)
}
{% endhighlight %}

* I don't like `UITableViewController` because of my bad experience with it in XCode 3. So I start with an `UIViewController` with a `tableView` in it. With above protocols defined, the controller will looks like this

{% highlight swift %}
class ViewController: UIViewController, UITableViewDelegate, UITableViewDataSource {
@IBOutlet weak var tableView: UITableView!
    var dataSource: [CustomDataSource1]!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
        CustomDataSource1.register(tableView: tableView)
        tableView.delegate = self
        tableView.dataSource = self
        tableView.separatorStyle = .none
        
        //clear empty cell
        tableView.tableFooterView = UIView()
    }

    // MARK: - TableView
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return dataSource.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let content = dataSource[indexPath.row]
        let cell = tableView.dequeueReusableCell(withIdentifier: content.cellIdentifier())!
        
        if let signUpCell = cell as? BECellRender {
            signUpCell.renderCell(data: content)
        }
        
        return cell
    }
 }
{% endhighlight %}

As you can see, all the code can be reusable apart from the `dataSource`. In the first thought, I can create a class with generic type for our base controller. But I will deal with that later. I move on our models first to see how it works.

{% highlight swift %}
struct CustomDataSource1: BECellDataSource {
    enum cellType {
        case type1
        case type2
        
        func cellIdentifier() -> String {
            switch self {
            case .type1:
                return "CustomDataSource1Type1"
            default:
                return "CustomDataSource2Type2"
            }
        }
        
        func nib() -> UINib {
            switch self {
            case .type1:
                return UINib(nibName: "CustomDataSource1Type1", bundle: nil)
            default:
                return UINib(nibName: "CustomDataSource1Type2", bundle: nil)
            }
        }
    }
    
    let type: cellType
    
    func cellIdentifier() -> String {
        return type.cellIdentifier()
    }
    
    static func register(tableView: UITableView) {
        tableView.register(cellType.type2.nib(), forCellReuseIdentifier: cellType.type2.cellIdentifier())
        tableView.register(cellType.type1.nib(), forCellReuseIdentifier: cellType.type1.cellIdentifier())
    }
}

class CustomDataSource1Type1: UITableViewCell, BECellRender {

    override func awakeFromNib() {
        super.awakeFromNib()
        // Initialization code
    }
    
    func renderCell(data: BECellDataSource) {
        if let d = data as? CustomDataSource1 {
            // start render your cell here
        }
    }
}
{% endhighlight %}

Build and run, you will see our cells are rendered on the table. Great :).

I have defined a pattern to implement tables. But it isn't optimal and we still need to do a lot of work for every table. So I keep going on.
First look at the custom cell, I really don't like the idea of the `render(data:)` function. It's ugly and if you have many cell classes for many kind of dataSource, it will be worser. So I modify the cell protocol a little bit
{% highlight swift %}
protocol BECellRenderImpl: BECellRender {
    associatedtype CellData
    func renderCell(data: CellData)
}

extension BECellRender where Self: BECellRenderImpl {
    func renderCell(data: BECellDataSource) {
        if let d = data as? CellData {
            renderCell(data: d)
        }
    }
}

{% endhighlight %} 

With the new protocol, I can specify the type of dataSource that will be used in the cell. Look at the new cell implementation: 
{% highlight swift %}
class CustomDataSource1Type2: UITableViewCell, BECellRenderImpl {

    typealias CellData = CustomDataSource1
    override func awakeFromNib() {
        super.awakeFromNib()
        // Initialization code
    }

    func renderCell(data: CustomDataSource1) {
        //Start implementing your cell
    }
}
{% endhighlight %}
It is far better now. I can say this cell is used with `CustomDataSource1` model. 

-

We have done a lot of work. From now on, we just create our `dataSource` model and specify which custom it will use. Then in the custom cell, implement the `renderCell(data:)`. That's all for the models. The latest step is in view controller. It will looks exactly the same as the view controller we defined above with another kind of dataSource. 

But, wait, if we are using the same code for our view controller, why don't we generic it? So let say, we will define a base view controller with a generic type like this:
{% highlight swift %}
	class BEBaseViewController<T : BECellDataSource>: UIViewController, UITableViewDataSource, UITableViewDelegate {
    var dataSource: [T] = []
    var tableView: UITableView!
    override func viewDidLoad() {
        super.viewDidLoad()

        // Do any additional setup after loading the view.
        T.register(tableView: tableView)
        tableView.delegate = self
        tableView.dataSource = self
        tableView.separatorStyle = .none
        
        //clear empty cell
        tableView.tableFooterView = UIView()
        
        tableView.reloadData()
    }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        // Dispose of any resources that can be recreated.
    }

    // MARK: - TableView
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return dataSource.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let content = dataSource[indexPath.row]
        let cell = tableView.dequeueReusableCell(withIdentifier: content.cellIdentifier())!
        
        if let signUpCell = cell as? BECellRender {
            signUpCell.renderCell(data: content)
        }
        
        return cell
    }
}
{% endhighlight %}

Now, we can rewrite our previous view controller just like this:

```
class ViewController: BEBaseViewController<CustomDataSource1> {
    
}
```
Woa, it is so simple to implement our table now. We just prepare the models and create a concrete view controller with that type of model. Then everything is in place. That's wonderful.

But, unfortunately, this approach doesn't work. Actually, it doesn't work with xib and storyboard. It will crash the app is you load the `ViewController` from a xib or storyboard :(. You can check [why generic class doesn't work with interface builder](https://stackoverflow.com/questions/25263882/use-a-generic-class-as-a-custom-view-in-interface-builder). I hope the swift team will fix this issue in a near future, but for now, we are limited to implement the view controller manually.

So, I can't have a perfect solution for what I want, but it is still worth to use in my project. I still need to re-implement the `UITableViewDataSource` for every view controller, but it can be easily done using code snippet because the code is exactly the same.

If you have any idea, please share. I'd love to hear from you!

You can check all the code in this [repository](https://github.com/sahara108/BlogExample-AssociatedType)