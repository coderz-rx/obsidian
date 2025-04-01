1、创建数据源 DataSource
2、为mybatis设置SqlSessionFactory（在这里绑定mapper.xml扫描路径，同时配置数据源DataSource）
3、使用@MapperScan(basePackages = "com.qunar.hotel.qhotel.goldcoins.repository", sqlSessionFactoryRef = "sugarSessionFactory")注解在数据源创建类上绑定mapper service文件扫描地址。（其中sugarSessionFactory为创建的SqlSessionFactory的bean名称）