# View State Transitions
+ 当ViewState从一种状态转换为另一种状态时，ViewState转换提供了平滑且具有视觉吸引力的转换。使用`Deck's viewState`属性进行转换。
+ 通过`viewState`的以下字段可以达到viewport的转换  
    - `transitionDuration` (Number|String, optional, default: 0) - 转换持续时间(毫秒)，默认值0，表示没有转换。当使用`FlyToInterpolator`时，可以设置为`auto`，根据开始和结束视图自动计算实际持续时间，并与它们之间的距离成线性。可以使用`FlyToInterpolator`构造函数的`speed`参数进一步定制持续时间（duration）。
    - `transitionEasing` (Function, optional, default: t => t) - Easing function that can be used to achieve effects like "Ease-In-Cubic", "Ease-Out-Cubic", etc. Default value performs Linear easing. (list of sample easing functions: http://easings.net/)
    - `transitionInterpolator` (Object, optional, default: LinearInterpolator) - 一个内插器对象，它定义了两个视图(viewport)之间的转换行为,deck.gl提供了`LinearInterpolator`和`FlyToInterpolator`。默认值是`LinearInterpolator`，在`ViewState`字段之间执行线性插值。`FlyToInterpolator`应用于`MapState`，使`ViewStates`的变换类似于MapBox的`flyto`API，当相机中心进行很长距离的变化时非常适用。用户也可以使用`TrasitionInterpolator`基类为这个对象提供任何自定义实现。
    - `transitionInterruption` (TRANSITION_EVENTS (Number), optional, default: BREAK) - 该字段控制如何处理在执行现有转换时发生的新`ViewState`更改。转换一旦执行完毕，该字段就不再起作用。下面列出了所有可能的值和结果行为。  
        |TRANSITION_EVENTS|Result|
        |---|---|
        |BREAK|当前转换将在当前状态停止，处理下一个`ViewState`更新。|
        |SNAP_TO_END|当前转换将跳过剩余的转换步骤，`ViewState`将更新为最终值，转换将停止，并处理下一个`ViewState`更新。|
        |IGNORE|在当前转换完成之前，任何`ViewState`更新都将被忽略，这还包括由于用户交互而导致的`ViewState`更改。|
    - `onTransitionStart` (Functional, optional) - 当请求的转换开始时触发回调.
    - `onTransitionInterrupt` (Functional, optional) - 在转换被中断时触发回调。
    - `onTransitionEnd` (Functional, optional) - 在转换结束时触发回调。
+ Usage
    - 示例代码提供了`flyTo`样式转换，以将摄像机从当前位置移动到纽约市。
        ```es6
        import DeckGL, {FlyToInterpolator} from 'deck.gl';
        import {StaticMap} from 'react-map-gl';

        class App extends Component {
        constructor(props) {
            super(props);
            this.state = {
            viewState: {
                latitude: 37.7751,
                longitude: -122.4193,
                zoom: 11,
                bearing: 0,
                pitch: 0,
                width: 500,
                height: 500
            }
            };
            this._onViewStateChange = this._onViewStateChange.bind(this);
        }

        _goToNYC() {
            this.setState({
            viewState: {
                ...this.state.viewState,
                longitude: -74.1,
                latitude: 40.7,
                zoom: 14,
                pitch: 0,
                bearing: 0,
                transitionDuration: 8000,
                transitionInterpolator: new FlyToInterpolator()
            }
            });
        }

        _onViewStateChange({viewState}) {
            this.setState({viewState});
        }

        render() {
            const {viewState} = this.state;

            return (
            <div>
                <DeckGL
                viewState={viewState}
                controller={MapController}
                onViewStateChange={this._onViewStateChange}
                >
                <StaticMap
                    // props
                    ...
                />
                </DeckGL>

                <button onClick={this._goToNYC}>New York City</button>
            </div>
            );
        }
        }
        ```
    - 样例代码实现沿垂直轴的连续旋转，直到用户通过鼠标交互旋转地图而中断。它使用`LinearInterpolator`并限制`bering`的转换。连续转换是通过使用`onTranstionEnd`回调触发新的转换来实现的。
        ```es6
        import DeckGL from 'deck.gl';
        import {StaticMap} from 'react-map-gl';

        const transitionInterpolator = new LinearInterpolator(['bearing']);

        const INITIAL_VIEW_STATE = {
        // set to required initial view state
        ...
        };

        class App extends Component {
        constructor(props) {
            super(props);
            this.rotationStep = 0;
            this.state = {
            viewState: INITIAL_VIEW_STATE
            };

            this._onLoad = this._onLoad.bind(this);
            this._onViewStateChange = this._onViewStateChange.bind(this);
            this._rotateCamera = this._rotateCamera.bind(this);
        }

        _onLoad() {
            this._rotateCamera();
        }

        _onViewStateChange({viewState}) {
            this.setState({viewState});
        }

        _rotateCamera() {
            // change bearing by 120 degrees.
            const bearing = this.state.viewState.bearing + 120;
            this.setState({
            viewState: {
                ...this.state.viewState,
                bearing,
                transitionDuration: 1000,
                transitionInterpolator,
                onTransitionEnd: this._rotateCamera
            }
            });
        }

        _renderLayers() {
            // render any deck.gl layers
            ...
        }

        render() {
            const {viewState} = this.state;
            return (
            <DeckGL
                layers={this._renderLayers()}
                viewState={viewState}
                onLoad={this._onLoad}
                onViewStateChange={this._onViewStateChange}
                controller={true}
            >
                <StaticMap
                // props
                ...
                />
            </DeckGL>
            );
        }
        }
        ```
# TransitionInterpolator
基本插值器类（Base interpolator class），为两个`ViewState`属性之间的转换提供公共功能。该类能被子类化来实现自定义插值。
+ Constructor  

    Parameters:
    - opts (Object | Array) - Object with following fields
        - `compare`: prop names used in equality check.
        - `extract`: prop names needed for interpolation.
        - `required`: prop names that must be supplied.
        - Array of prop names that are used for all above fields.    
+ Methods
    - **arePropsEqual**：判断两个`ViewState`的异同  

        Parameters:
        - `currentProps`: Object with ViewState props.
        - `nextProps`: Object with ViewState props.  

        Returns:
        - `true` if the ViewStates have equal value for all `compare` props.
    - **initializeProps**  

        Parameters:
        - `startProps` (Object): Object with staring ViewState props.
        - `endProps` (Object): Object with ending ViewState props.

        Returns:
        - {start, end}, transition props are validated and extracted from inputs and returned.

    - **interpolateProps**
        - 此方法未实现，必须由子类实现。
# LinearInterpolator
插值类(Interpolator class)，继承自`TransitionInterpolator`。执行两个视图状态之间的线性插值。
+ Constructor

    Parameters:  
    - transitionProps (Array, default: ['longitude', 'latitude', 'zoom', 'bearing', 'pitch']): Array of props that are linearly interpolated.
+ Methods
    - **interpolateProps**

        Parameters:
        - `startProps` (Object): Object with staring ViewState props.
        - `endProps` (Object): Object with ending ViewState props.
        - `t` (Number) : Number in [0, 1] range.

        Returns:
        - Object with interpolated ViewState props.
# FlyToInterpolator
插值类(Interpolator class)，继承自`TransitionInterpolator`。这个类被设计成在两个`MapState`对象之间执行`flyTo`样式的插值。
+ Constructor

    Parameters:
    - props (Object) - Object with following fields
        - `curve` (Number, optional, default: 1.414) - The zooming "curve" that will occur along the flight path.沿着航线进行伸缩变换。
        - `speed` (Number, optional, default: 1.2) - 与`option.curve`相关的动画的平均速度。它线性地影响持续时间，更高的速度返回更小的持续时间，反之亦然。
        - `screenSpeed` (Number, optional) - 动画的平均速度以每秒屏幕大小来衡量。与`opts.speed`相似，当忽略指定的`opts.speed`时，它会线性地影响持续时间。
        - `maxDuration` (Number, optional) - 最大持续时间(以毫秒为单位)，如果计算的持续时间超过此值，则返回0。

    - Initializes super class with an object with following props:

        - compare: ['longitude', 'latitude', 'zoom', 'bearing', 'pitch']
        - extract: ['width', 'height', 'longitude', 'latitude', 'zoom', 'bearing', 'pitch']
        - required: ['width', 'height', 'latitude', 'longitude', 'zoom']
+ Methods
    - **interpolateProps**

        Parameters:
        - `startProps` (Object): Object with staring ViewState props.
        - `endProps` (Object): Object with ending ViewState props.
        - `t` (Number) : Number in [0, 1] range.

        Returns:
        - Object with interpolated ViewState props.
    - **getDuration**

        Parameters:
        - `startProps` (Object): Object with staring ViewState props.
        - `endProps` (Object): Object with ending ViewState props.

        Returns:
        - `transitionDuration` value in milliseconds.