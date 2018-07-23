## state和props

### state

与组件相关显示相关的属性放在状态中
初始化有两种方式，一是在构造方法里，二是直接在类中
```JavaScript
class Video extends React.Component {
    state = {
        loopsRemaining: this.props.maxLoops,
    }
}
//推荐更易理解的在构造函数中初始化（这样你还可以根据需要做一些计算）：
class Video extends React.Component {
    constructor(props){
        super(props);
        this.state = {
            loopsRemaining: this.props.maxLoops,
        };
    }
}
```

### props
父视图传递下来的信息放在属性中

### 设置默认属性和属性的类型
```JavaScript
class Video extends React.Component {
    static defaultProps = {
        autoPlay: false,
        maxLoops: 10,
    };  // 注意这里有分号
    static propTypes = {
        autoPlay: React.PropTypes.bool.isRequired,
        maxLoops: React.PropTypes.number.isRequired,
        posterFrameSrc: React.PropTypes.string.isRequired,
        videoSrc: React.PropTypes.string.isRequired,
    };  // 注意这里有分号
    render() {
        return (
            <View />
        );
    } // 注意这里既没有分号也没有逗号
}
```