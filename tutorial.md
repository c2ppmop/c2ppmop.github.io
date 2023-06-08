## **Tutorials**

We provide two example for using **CCMOP**. We first use **CCMOP** to analyze a single file with respect to the Safe-Iterator property. Then we use **CCMOP** to analyze a real-world project for the Use-after-free property.  

## **Run the docker**
We suppose that you have installed the docker image named **ccmop/ccmop**; otherwise please go to [Manual](https://rv-ccmop.github.io/manual) for help. You can run the following command to get a container if you have not created the container from the image named **ccmop/ccmop**.
```shell
docker run ccmop/ccmop
```
After the above command, you can run the following command to have a check.
```shell
docker ps -a
```
If the creation of the container is successful, you will see a container created from the image named **ccmop/ccmop**. Then the following commands should be run to get into the container.
```shell
docker start <CONTAINER ID>
docker exec -it <CONTAINER ID> /bin/bash
```
## **The single-file Example**

The first example shows the verification where the property is Safe-Iterator. The property of Safe-Iterator is as follows.  
- Event **create** means get an iterator from a string.  
- Event **update** means update the string.  
- Event **derefIterator** means the dereference of an iterator, which is an operator call.  
- **within(main())** will limit the matching scope to the main function that has no effect in this example.   
```code
#include<iostream>
UnSafeIterator(std::string s, std::string::iterator it){
     event create after(std::string s, std::string::iterator it):
    (call(iterator string.begin())||call(iterator string.end()))
     &&result(it) && target(s) &&within(main()){}
     
    event update after(std::string s) :
    (call(% string.clear(...))
    ||call(% string.insert(...))
    ||call(% string.erase(...))
    ||call(% string.push_back(...))
    ||call(% string.pop_back(...))
    ||call(% string.operator=(...))
    ||call(% string.operator+=(...))) && target(s) {}
    
    event derefIterator before(std::string::iterator it):call(% operator*())&&args(it){}
    
    ere : create derefIterator* update update* derefIterator
    
        @match {
        std::cout<<"improper iterator usage!"<<std::endl;
        }
}
```

The example's source code for **Safe-Iterator** verification is as follows.  

```code
#include <iostream>
int main() {
    std::string s = "hello";
    std::string::iterator it=s.begin();
    s.clear();
    std::cout<<*it<<std::endl;
    return 0;
};
```
We use the following command to compile the above example.

```shell
cd /root/CCMOP/examples/CXX/UnSafeIterator
wac  -cxx -print -mop ./UnSafeIterator.mop test.cpp -o example
./example
```

After instrumenting, the example's source code for **Safe-Iterator** is as follows.  

```code
#include <iostream>
int main() {
    std::string s = "hello";
    std::string::iterator it = s.begin();
    std::string &s_2 = s;
    std::string::iterator &it_2 = it;
    __RVC_UnSafeIterator_create(s_2, it_2, "test.cpp:4:30");
    std::string &s_1 = s;
    s.clear();
    __RVC_UnSafeIterator_update(s_1, "test.cpp:5:5");
    std::string::iterator &it_3 = it;
    __RVC_UnSafeIterator_derefIterator(it_3, "test.cpp:6:16");
    std::cout << * it << std::endl;
    return 0;
}
```
After running the generated binary, we could see the verification results of the example.  
The **s** and **it** in the result are the id from the property.

```code
s address:0x7ffdbb6d9ca8
it address:0x7ffdbb6d9c88
create test.cpp:4:30
|
update test.cpp:5:5
|
derefIterator test.cpp:6:16
|
V
improper iterator usage!

```
## **A Project Example**
We use a  project named **jsoncpp**; the property is **Use-after-free**.  
- Event **create** will match all the new Expr in programs.  
- Event **free** will match all the delete Expr in programs.  
- Event **deref** will match all the pointer dereference in programs that pass the dereferenced pointer as key default.  

```code
#include<iostream>
UAF(void *key){
    event create after(void* key) : expr(new *(...))&& result(key) {}
    event free before(void* key) : expr(delete *(...))&& args(key){}

    event deref before(void *key):derefPointer(){}

    ere : create deref* free (create deref* free )* deref

    @ match {
        std::cout<<"use after free at address "<<std::endl;
         __RESET;
    }
}
```

To verify the target project with respect to the **Use-after-free** property, we need to create a **JSON** file containing the target project configuration as follows.

```json
{
"project_name":"jsoncpp",
"compile_language":"cxx",
"project_root_dir":"./",
"mop_file":"../../examples/CXX/UAF.mop",
"project_build_subdir":"build",
"project_compile_cmd":"cmake ..;make"
}
```

- We use a relative path related to this **JSON** file to represent the **project_root_dir** and **mop_file**.  

We use the following command to compile the target project.  
```shell
cd /root/benchmarks/CXX/jsoncpp
wac -proj test_build.json
```
In compilation, CCMOP will first generate a runtime library for property in **project_root_dir/UAFaspect**. Then CCMOP will enter the **project_build_subdir** and run the **project_compile_cmd**.
If the compilation of target project is successful, you could get in the **project_build_subdir** and have a test.
```shell
cd build
./bin/jsoncpp_test
```
After above commands, you will see the results in bash as shown as below.
```code
...
...
...
Testing IteratorTest/distance: OK
Testing IteratorTest/nullValues: OK
Testing IteratorTest/staticStringKey: OK
Testing IteratorTest/names: OK
Testing IteratorTest/indexes: OK
Testing IteratorTest/constness: key address:0x557f8332ec40
create /root/benchmarks/CXX/jsoncpp/src/test_lib_json/main.cpp:3763:1 <Spelling=<scratch space>:138:1>
|
free /root/benchmarks/CXX/jsoncpp/src/test_lib_json/jsontest.cpp:251:3
|
create /root/benchmarks/CXX/jsoncpp/src/test_lib_json/main.cpp:3776:1 <Spelling=<scratch space>:156:1>
|
free /root/benchmarks/CXX/jsoncpp/src/test_lib_json/jsontest.cpp:251:3
|
create /root/benchmarks/CXX/jsoncpp/src/test_lib_json/main.cpp:3797:1 <Spelling=<scratch space>:9:1>
|
free /root/benchmarks/CXX/jsoncpp/src/test_lib_json/jsontest.cpp:251:3
|
create /root/benchmarks/CXX/jsoncpp/src/test_lib_json/main.cpp:3817:1 <Spelling=<scratch space>:36:1>
|
free /root/benchmarks/CXX/jsoncpp/src/test_lib_json/jsontest.cpp:251:3
|
create /root/benchmarks/CXX/jsoncpp/src/test_lib_json/main.cpp:3823:1 <Spelling=<scratch space>:55:1>
|
free /root/benchmarks/CXX/jsoncpp/src/test_lib_json/jsontest.cpp:251:3
|
deref /root/benchmarks/CXX/jsoncpp/src/lib_json/json_value.cpp:133:3
|
V
use after free at address 0x557f8332ec40
OK
Testing RValueTest/moveConstruction: OK
Testing FuzzTest/fuzzDoesntCrash: OK
Testing MemberTemplateAs/BehavesSameAsNamedAs: OK
Testing MemberTemplateIs/BehavesSameAsNamedIs: OK
Testing VersionTest/VersionNumbersMatch: OK
All 119 tests passed
```

# [](#header-1)**Contacts**

Please feel free to contact us if you have any questions about **CCMOP**.

*   <font color="#0000FF" size="4">Yongchao Xing (xingyc0979@nudt.edu.cn)</font>

*   <font color="#0000FF" size="4"> Zhenbang Chen (zbchen@nudt.edu.cn)</font>
