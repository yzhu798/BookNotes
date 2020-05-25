# 一、Install

## 1.Tools

```bash
sudo apt-get install cmake git lcov xdg-utils
```

## 2.Libs-dev

```bash
sudo apt-get install libssl-dev uuid-dev libmosquitto-dev libgtest-dev
```

## 3.MQTT server (mosquitto)

```bash
sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
sudo apt-get update
sudo apt-get install mosquitto
```

## 4.MQTT Client (paho.mqtt.c)

```bash
git clone https://github.com/eclipse/paho.mqtt.c.git
cd paho.mqtt.c
make
sudo make install
```

# 二、Configuration

## 1.CMakeList.txt

```cmake
cmake_minimum_required(VERSION 3.4.3)
project(rf5800)
set(BINARY_NAME "rf5800")

if(CMAKE_BUILD_TYPE MATCHES Debug)
    set(BINARY_NAME "rf5800Unit")
endif()
#################     COMMAND git
execute_process(
    COMMAND git rev-parse --abbrev-ref HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_BRANCH
    OUTPUT_STRIP_TRAILING_WHITESPACE
    )

execute_process(
    COMMAND git log -1 --pretty=format:%cd\ %cn:%s --date=iso
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_COMMIT_MESSAGE
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
execute_process(
    COMMAND git rev-parse --short=10 HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_COMMIT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

message("GIT_BRANCH: " ${GIT_BRANCH})
message("GIT_COMMIT_HASH: " ${GIT_COMMIT_HASH})
message("GIT_COMMIT_MESSAGE: " ${GIT_COMMIT_MESSAGE})

add_definitions(-DVERSION="g${GIT_COMMIT_HASH}")
add_definitions(-D"USE_HRSLEEP")
add_definitions(-DGCP_DEBUG_SOURCE)
add_definitions(-DGCPOS_LINUX)
add_definitions(-D"OS_LINUX")
#################     COMMAND git
set(CMAKE_C_STANDARD 90)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ggdb3")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fmessage-length=0")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-psabi")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror=return-type")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fstack-protector-all")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror=format")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror=write-strings")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror=reorder")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror=unused-variable ")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fpermissive ")

if(CMAKE_BUILD_TYPE MATCHES Debug)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage ")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs ")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ftest-coverage ")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-access-control ")
set(CMAKE_EXE_LINKER_FLAGS -fprofile-arcs  -ftest-coverage) 
endif()

set(CMAKE_EXE_LINKER_FLAGS -lssl)
set(CMAKE_EXE_LINKER_FLAGS -lcrypto)
set(CMAKE_EXE_LINKER_FLAGS -ldl)

file(GLOB_RECURSE SOURCES_LINUX_UTILITY
	${CMAKE_CURRENT_SOURCE_DIR}/Source/linuxutility/*.cpp
	)	

add_library(linuxutility ${SOURCES_LINUX_UTILITY})

file(GLOB_RECURSE SOURCES_QMODEL
	${CMAKE_CURRENT_SOURCE_DIR}/qmodel/*.cpp
	)	

add_library(qmodel ${SOURCES_QMODEL})

file(GLOB_RECURSE SOURCES_COMMON
	${CMAKE_CURRENT_SOURCE_DIR}/Common/*.cpp
	)	

add_library(Common ${SOURCES_COMMON})


file(GLOB_RECURSE SOURCES_MAIN
	${CMAKE_CURRENT_SOURCE_DIR}/Source/HW5800BusController/*.cpp
	${CMAKE_CURRENT_SOURCE_DIR}/Source/HW5800BusController/System/*.cpp
    )
if(CMAKE_BUILD_TYPE MATCHES Debug)

file(GLOB_RECURSE SOURCES_MAIN
	${CMAKE_CURRENT_SOURCE_DIR}/unitTest/*.cpp
	${CMAKE_CURRENT_SOURCE_DIR}/main/main.cpp
    )
else()
file(GLOB_RECURSE SOURCES_MAIN
	${CMAKE_CURRENT_SOURCE_DIR}/Source/Main.cpp
    )
endif()

include_directories(
	${CMAKE_CURRENT_SOURCE_DIR}/Source/
	${CMAKE_CURRENT_SOURCE_DIR}/Source/Buscontroller
	${CMAKE_CURRENT_SOURCE_DIR}/Source/Device
	${CMAKE_CURRENT_SOURCE_DIR}/Source/Keypad
	${CMAKE_CURRENT_SOURCE_DIR}/Source/Keypad/Types
	${CMAKE_CURRENT_SOURCE_DIR}/Source/linuxutility/
	${CMAKE_CURRENT_SOURCE_DIR}/Common/
	${CMAKE_CURRENT_SOURCE_DIR}/qmodel/
	)

if(CMAKE_BUILD_TYPE MATCHES Debug)
include_directories(
	${CMAKE_CURRENT_SOURCE_DIR}/unitTest/
	)
endif()

add_executable(${BINARY_NAME} ${SOURCES_MAIN})

find_library(LIBDL NAMES dl)
find_library(LIBPTHREAD NAMES pthread)
find_library(LIBRT NAMES rt)
find_library(LIBCRYPTO NAMES crypto)
find_library(LIBMOSQUITTOPP names mosquitto)

find_library(LIBMQTTAS NAMES paho-mqtt3as)
find_library(LIBMQTTA NAMES paho-mqtt3a)
find_library(LIBMQTTCS NAMES paho-mqtt3cs)
find_library(LIBMQTTC NAMES paho-mqtt3c)

if(CMAKE_BUILD_TYPE MATCHES Debug)
####  find_library(LIBGTESTMAIN NAMES gtest_main)
find_library(LIBGTEST NAMES gtest)
find_library(LIBGMOCK NAMES gmock)
endif()
target_link_libraries(${BINARY_NAME}
    linuxutility
	Common
	qmodel
    ${LIBRT}
    ${LIBDL}
    ${LIBCRYPTO}
    ${LIBPTHREAD}
	paho-mqtt3as 
	paho-mqtt3c 
	paho-mqtt3cs 
	paho-mqtt3a
    uuid
	)
if(CMAKE_BUILD_TYPE MATCHES Debug)
target_link_libraries(${BINARY_NAME}
#    ${LIBGTESTMAIN}
    ${LIBGTEST}
	${LIBGMOCK}
	
	)
endif()
install(TARGETS ${BINARY_NAME}
        DESTINATION /honeywell/app/${BINARY_NAME}
       )

add_dependencies(${BINARY_NAME} linuxutility)
add_dependencies(${BINARY_NAME} qmodel)

set(MAJOR_VERSION "1")
set(MINOR_VERSION "0")
set(PATCH_VERSION "0")
set(CPACK_GENERATOR "DEB")
set(CPACK_PACKAGE_NAME "${BINARY_NAME}_${GIT_BRANCH}-${GIT_COMMIT_HASH}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY ${GIT_COMMIT_MESSAGE})
set(CPACK_PACKAGE_CONTACT "Honeywell")
set(CPACK_PACKAGE_VERSION ${MAJOR_VERSION}.${MINOR_VERSION}.${PATCH_VERSION}+git0+${GIT_COMMIT_HASH}+${GIT_BRANCH}-r0)
set(CPACK_PACKAGE_VERSION_MAJOR "${MAJOR_VERSION}")
set(CPACK_PACKAGE_VERSION_MINOR "${MINOR_VERSION}")
set(CPACK_PACKAGE_VERSION_PATCH "${PATCH_VERSION}")
set(CPACK_PACKAGE_ARCHITECTURE "armv7ahf-neon")
set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_ARCHITECTURE})
set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE ${CPACK_PACKAGE_ARCHITECTURE})
set(CPACK_DEBIAN_PACKAGE_SECTION "honeywell")
set(CPACK_DEBIAN_ARCHIVE_TYPE "paxr")
set(CPACK_DEBIAN_COMPRESSION_TYPE "gzip")
set(CPACK_SET_DESTDIR true)
set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_BINARY_DIR}")
set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
set(CPACK_STRIP_FILES "EXECUTABLE_OUTPUT_PATH/${BINARY_NAME}")
include(CPack)
```

# 三、Build & Run

## 1. ARM

### Build ipk package

```bash
#!/bin/bash
git submodule update --init --remote
shopt -s expand_aliases
rm build -rf 
mkdir -p build
cd build
rm *.ipk
let "CPU_COUNT = `nproc` - 1"
source /opt/pancake-core-sdk/environment-setup-armv7ahf-neon-poky-linux-gnueabi  
#cmake & make package
cmake -DCMAKE_BUILD_TYPE = NORMAL .. && make package -j $CPU_COUNT
echo "rename deb to ipk"
for file in *.deb; do mv -- "${file}" "${file/%deb/ipk}"; done
```

## 2. X86

### RunUnit

```bash
#!/bin/bash
current_path     =`pwd`
# set unitExecutefile
unitExecutefile  = rf5800Unit  
rm -rf unitTestResult
mkdir -p unitTestResult/gcov_source/

sudo chmod +x /build/$unitExecutefile
sudo ./build/$unitExecutefile --gtest_output = xml:unitTestResult/ --gtest_print_time
```

### Analyse

```bash
#!/bin/bash
current_path     =`pwd`
# set unitExecutefile
unitExecutefile  = rf5800Unit
unitInfoFile     =${unitExecutefile}.info
rm -rf analyseFile
mkdir ./analyseFile/
rm unitInfoFile

#to generate gcov files and cp them to unitTestResult/gcov_source/
gcov_path=./build/CMakeFiles/$unitExecutefile.dir/Source
for dir in $gcov_path/*
do
	if [ -d $dir ]
	then
		cd $dir
		gcov *.cpp.o
	fi
done
cd $current_path/$gcov_path
gcov *.cpp.o
cd $current_path
echo ======================generate lcov info ======================
find . -name "*.gcno" |xargs  -t -i mv {} ./analyseFile/
find . -name "*.gcda" |xargs  -t -i mv {} ./analyseFile/
#   analyseFile
cd analyseFile
gcov *.gcno
cd $current_path
find . -name "*.gcov" |xargs  -t -i mv {} unitTestResult/gcov_source/
find . -name "*.gcno" |xargs  -t -i mv {} unitTestResult/gcov_source/
find . -name "*.gcda" |xargs  -t -i mv {} unitTestResult/gcov_source/
find . -name "${unitExecutefile}.xml" |xargs  -t -i mv {} unitTestResult/${unitExecutefile}.xml

echo ======================Lcov Analyse Start ======================
lcov -d ./analyseFile/ -t ${unitExecutefile} -o ${unitExecutefile}.info -b . -c --rc lcov_branch_coverage=1

lcov --remove ${unitExecutefile}.info '/usr/*' -o ${unitExecutefile}.info
echo ======================Create Web and Sheet ====================
genhtml  --rc genhtml_branch_coverage=1 -o unitTestResult ${unitExecutefile}.info

python3 gtest_xml_convert.py unitTestResult/${unitExecutefile}.xml unitTestResult/result.xml

```

### Kill mkpty.py

```bash
#!/bin/bash
sudo ps -ef |grep mkpty.py |awk '{print $2}'|xargs kill -9
python3 mkpty.py &
sudo ps -ef |grep mkpty.py |awk '{print $2}'|xargs kill -9
```

### Clear

```bash
#!/bin/bash
rm -rf analyseFile
rm -rf UnitTestResult
rm -rf 5800UnitTest.info
rm -rf build
```

### Build X86_bin

```bash
#!/bin/bash
git submodule update --init --remote
shopt -s expand_aliases
rm -rf build
mkdir -p build
cd build
let "CPU_COUNT = `nproc` - 1"
#cmake & make
cmake -DCMAKE_BUILD_TYPE = Debug .. && make  -j $CPU_COUNT
```

### convert result

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
# file gtest_xml_convert.py
import datetime
import sys
import os
import io
from xml.dom.minidom import parse
import xml.dom.minidom

class TestSuites:
    def __init__(self):
        self.name = ""
        self.tests = 0
        self.failures = 0
        self.time = 0.0
        self.timestamp = ""
        self.testsuites = []

class TestSuite:
    def __init__(self):
        self.name = ""
        self.tests = 0
        self.failures = 0
        self.time = 0.0
        self.timestamp = ""
        self.testcases = []

class TestCase:
    def __init__(self):
        self.name = ""
        self.status = ""
        self.result = ""
        self.time = 0.0
        self.timestamp = ""
        self.classname = ""
        self.failures = []

g_testresult = TestSuites()

def useage():
    print("Useage:")
    print("python " + sys.argv[0] + " {input xml name}" + " {output xml name}")

def parse_xml(filename):
    if (not os.path.isfile(filename)):
        print(filename + " not a exsit file!")
        return

    xml_tree = parse(filename);
    root_node = xml_tree.documentElement;
    print(root_node.nodeName);
    if (root_node.nodeName == 'testsuites'):
        print("begin to parse file " + filename)
        g_testresult.name = root_node.getAttribute('name')
        g_testresult.tests = int(root_node.getAttribute('tests'))
        g_testresult.failures = int(root_node.getAttribute('failures'))
        g_testresult.time = float(root_node.getAttribute('time'))
        g_testresult.timestamp = root_node.getAttribute('timestamp')
        testsuites = root_node.getElementsByTagName('testsuite')
        for testsuite in testsuites:
            testsuite_inst = TestSuite()
            testsuite_inst.name = testsuite.getAttribute('name')
            testsuite_inst.tests = int(testsuite.getAttribute('tests'))
            testsuite_inst.failures = int(testsuite.getAttribute('failures'))
            testsuite_inst.time = float(testsuite.getAttribute('time'))
            testsuite_inst.timestamp = testsuite.getAttribute('timestamp')
            testcases = testsuite.getElementsByTagName('testcase')
            for testcase in testcases:
                testcase_inst = TestCase()
                testcase_inst.name = testcase.getAttribute('name')
                testcase_inst.status = testcase.getAttribute('status')
                testcase_inst.result = testcase.getAttribute('result')
                testcase_inst.time = float(testcase.getAttribute('time'))
                testcase_inst.timestamp = testcase.getAttribute('timestamp')
                testcase_inst.classname = testcase.getAttribute('classname')
                failures = testcase.getElementsByTagName('failure')
                for failure in failures:
                    failure_message = failure.getAttribute('message')
                    #failure_message.replace('&#x0A;', '')
                    print (failure_message)
                    testcase_inst.failures.append(failure_message)
                testsuite_inst.testcases.append(testcase_inst)
            g_testresult.testsuites.append(testsuite_inst)

def generate_xml(filename):
    print("begin to generate file " + filename)

    doc = xml.dom.minidom.Document()
    root_node = doc.createElement('testExecutions')
    root_node.setAttribute('version','1')
    doc.appendChild(root_node)

    file_node = doc.createElement('file')
    file_node.setAttribute('path','unitTest/Unittest.cpp')
    root_node.appendChild(file_node)
    for testsuite in g_testresult.testsuites:
        for testcase in testsuite.testcases:
            testcase_node = doc.createElement('testCase')
            testcase_node.setAttribute('name', testcase.name)
            testcase_node.setAttribute('duration', str(int(testcase.time * 1000)))
            for failure in testcase.failures:
                failure_node = doc.createElement('failure')
                failure_node.setAttribute('message', failure)
                failure_node.appendChild(doc.createTextNode(''))
                testcase_node.appendChild(failure_node)
            file_node.appendChild(testcase_node)

    #此处需要用codecs.open可以指定编码方式
    fp = open(filename, 'w')
    #将内存中的xml写入到文件
    doc.writexml(fp, indent='', addindent='\t', newl='\n', encoding='utf-8')
    fp.close()

def main():
    if (len(sys.argv) < 2):
        useage()
        exit()
    elif (sys.argv[1] == '-h' or sys.argv[1] == '-help'):
        useage()
        exit()
    elif (len(sys.argv) < 3):
        useage()
        exit()
    print('begin to process and xml:\n')
    print("begin to convert " + sys.argv[1] + " to " + sys.argv[2])
    parse_xml(sys.argv[1])
    generate_xml(sys.argv[2])

if __name__ == '__main__':
    main()
```

