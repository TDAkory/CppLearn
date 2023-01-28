# 关于精度

## Example 

```cpp
#include <iostream>
#include <iomanip>
#include <sstream>

using namespace std;

int main() {
    double num = 1642058085413901;

    std::stringstream metric1;
    metric1.setf(std::ios::fixed);  // 
    metric1 << "put ";
    metric1 << setprecision(6) << num;
    cout << metric1.str() << endl;      // put 1642058085413901.000000


    std::stringstream value_stream;
    value_stream << std::setprecision(6) << num;
    std::string metric;
    metric.append("put ");
    metric.append(value_stream.str()).append(" ");
    cout << metric << endl;             // put 1.64206e+15 

    return 0;
}

```

## [std::ios_base::setf](https://en.cppreference.com/w/cpp/io/ios_base/setf)

## [std::fixed](https://www.cplusplus.com/reference/ios/fixed/)

Sets the floatfield format flag for the str stream to fixed.

When floatfield is set to fixed, floating-point values are written using fixed-point notation: the value is represented with exactly as many digits in the decimal part as specified by the precision field (precision) and with no exponent part.

## std::ios::fixed

generate floating point types using fixed notation, or hex notation if combined with scientific: see [std::fixed](https://en.cppreference.com/w/cpp/io/manip/fixed)

## [setprecision](https://en.cppreference.com/w/cpp/io/manip/setprecision)

changes floating-point precision，6 by default
