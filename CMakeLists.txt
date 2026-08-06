cmake_minimum_required(VERSION 3.16)
project(deepin-liferaft CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

find_package(Dtk6Widget REQUIRED)

add_executable(deepin-liferaft main.cpp resources.qrc)
target_link_libraries(deepin-liferaft PRIVATE Dtk6::Widget)

enable_testing()
add_test(NAME self-test COMMAND deepin-liferaft --self-test)

install(TARGETS deepin-liferaft RUNTIME DESTINATION bin)
install(FILES deepin-liferaft.service DESTINATION lib/systemd/user)
#install(FILES deepin-liferaft.desktop DESTINATION share/applications)
install(FILES data/icons/deepin-liferaft.svg DESTINATION share/icons/hicolor/scalable/apps)
install(FILES deepin-liferaft.1 DESTINATION share/man/man1)
install(FILES README.md DESTINATION share/doc/deepin-liferaft)
