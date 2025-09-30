**_未完成_**

## 1、step 函数
  - 原型：Type step(Type edge, Type x)
  - 参数：无要求。
  - 返回值：等于 0.0 或者 1.0。
  - 作用：把 x 的值转为 0.0 或者 1.0。
    当 x < dege 时，函数返回 0.0；否则，函数返回 1.0。
  - 使用场景：
## 2、smoothstep 函数
  - 原型：Type smoothstep(Type edge0, Type edge1, Type x)
  - 参数：edge1 > edge0。
  - 返回值：在 0.0~1.0 之间，包括 0.0 和 1.0。
  - 作用：当 edge0 < x < edge1 时，smoothstep 在 0.0 和 1.0 之间执行平滑的插值。
  - 使用场景：
## 3、mix 函数
  - 原型：Type mix(Type x, Type y, Type a)
  - 参数：a 需要在 0.0~1.0 之间，包括 0.0 和 1.0。
  - 返回值：在 x~y 之间，包括 x 和 y。
  - 作用：根据 a 的分量，在 x 和 y 之间进行线性插值；a 等于 0.0，返回 x，a 等于 1.0，a 等于 y。
    效果等同于 x*(1-a)+y*a
  - 使用场景：
## 4、clamp 函数
  - 原型：Type clamp(Type x, Type minVal, Type maxVal)
  - 参数：无要求。
  - 返回值：在 minVal~maxVal 之间，包括 minVal 和 maxVal。
  - 作用：把 x 的值限制在 minVal~maxVal 之间。
    效果等同于 min(max(x, minVal), maxVal)。
  - 使用场景：
## 5、length与distance函数