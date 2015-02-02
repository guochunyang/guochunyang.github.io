---
layout: post
title: JDBC的封装
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
 
  连接数据库的步骤：
  1.加载JDBC驱动
  2.提供连接参数
  3.建立数据库连接
  4.创建一个statement
  5.执行SQL语句
  6.处理结果
  7.关闭JDBC对象
   
   
  新建一个JDBCUtil类
  

```Python
package com.jdbc.utils;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.ResultSetMetaData;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class JDBCUtil
{
    private final String USERNAME = "root"; //用户名
    private final String PASSWORD = "a123"; //密码
    private final String DRIVER = "com.mysql.jdbc.Driver"; //需要加载的驱动
    private final String URL = "jdbc:mysql://localhost:3306/test"; //访问数据库的url

    private Connection connection; //数据库连接
    private PreparedStatement preparedStatement;//定义sql语句的执行对象
    private ResultSet resultSet; //查询返回的结果集合


}
```
		

 


在构造函数中注册驱动:




```C++
public JDBCUtil()
    {
        try
        {
            Class.forName(DRIVER); //注册驱动
            System.out.println("注册成功");
        } catch (Exception e)
        {
            e.printStackTrace();
        }
    }
```
		

获取连接:




```C++
public Connection getConnection()
    {
        try
        {
            //获取连接
            connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);

        } catch (Exception e)
        {
            // TODO: handle exception
        }
        return connection;
    }
```
		

执行更新语句:




```C++
//执行更新语句
    public boolean updateByPreparedStatement(String sql, List<Object> params)
            throws SQLException
    {
        //获取sql执行对象
        preparedStatement = connection.prepareStatement(sql);
        //填充sql中的占位符
        if (params != null && !params.isEmpty())
        {
            for (int i = 0; i < params.size(); ++i)
            {
                preparedStatement.setObject(i + 1, params.get(i));
            }
        }
        
        //执行update
        int result = preparedStatement.executeUpdate();
        return result > 0;
    }
```
		

查询单条结果:




```C++
//查询单条结果
    public Map<String, Object> findSimpleResult(String sql, List<Object> params)
            throws SQLException
    {
        //map对应一条结果
        Map<String, Object> map = new HashMap<String, Object>();
        
        //填充占位符
        preparedStatement = connection.prepareStatement(sql);
        if (params != null && !params.isEmpty())
        {
            for (int i = 0; i < params.size(); ++i)
            {
                preparedStatement.setObject(i + 1, params.get(i));
            }
        }
        
        //执行sql语句
        resultSet = preparedStatement.executeQuery();
        //获取列的信息
        ResultSetMetaData metaData = resultSet.getMetaData();
        //获取列的个数
        int colLen = metaData.getColumnCount();

        while (resultSet.next())
        {
            //依次取出每列，放入map中
            for (int i = 0; i < colLen; ++i)
            {
                //获取列名
                String colsName = metaData.getColumnName(i + 1);
                //获取该列的value
                Object colsValue = resultSet.getObject(colsName);
                if (colsValue == null)
                {
                    colsValue = "";
                }
                //将该列放入map
                map.put(colsName, colsValue);
            }
        }

        return map;
    }
```
		

查询多条结果:




```C++
public List<Map<String, Object>> findMoreResult(String sql,
            List<Object> params) throws SQLException
    {
        //每个map对应一行数据
        List<Map<String, Object>> list = new ArrayList<Map<String, Object>>();
        
        preparedStatement = connection.prepareStatement(sql);
        if (params != null && !params.isEmpty())
        {
            for (int i = 0; i < params.size(); ++i)
            {
                preparedStatement.setObject(i + 1, params.get(i));
            }
        }

        resultSet = preparedStatement.executeQuery();
        ResultSetMetaData metaData = resultSet.getMetaData();
        int colLen = metaData.getColumnCount();

        //逐行进行遍历
        while (resultSet.next())
        {
            Map<String, Object> map = new HashMap<String, Object>();
            for (int i = 0; i < colLen; ++i)
            {
                String colsName = metaData.getColumnName(i + 1);
                Object colsValue = resultSet.getObject(colsName);
                if (colsValue == null)
                {
                    colsValue = "";
                }
                map.put(colsName, colsValue);
            }
            list.add(map);
        }

        return list;
    }
```
		

释放连接:




```C++
//释放连接
    public void releaseConn()
    {
        if (resultSet != null)
        {
            try
            {
                resultSet.close();
            } catch (Exception e)
            {
                // TODO: handle exception
                e.printStackTrace();
            }
        }

        if (preparedStatement != null)
        {
            try
            {
                preparedStatement.close();
            } catch (Exception e)
            {
                // TODO: handle exception
                e.printStackTrace();
            }
        }

        if (connection != null)
        {
            try
            {
                connection.close();
            } catch (Exception e)
            {
                // TODO: handle exception
                e.printStackTrace();
            }
        }
    }
```
		

 


 


测试代码如下:




```C++
public static void main(String[] args)
    {
        JDBCUtil jdbcUtil = new JDBCUtil();
        jdbcUtil.getConnection();

        String sql = "insert into person(name, age) values(?, ?)";
        List<Object> params = new ArrayList<Object>();
        params.add("rose");
        params.add(123);
        try
        {
            boolean flag = jdbcUtil.updateByPreparedStatement(sql, params);
            System.out.println(flag);
        } catch (Exception e)
        {
            // TODO: handle exception
            e.printStackTrace();
        }

        sql = "select * from person where id = 1";
        try
        {
            Map<String, Object> map = jdbcUtil.findSimpleResult(sql, null);
            System.out.println(map);
        } catch (Exception e)
        {
            // TODO: handle exception
            e.printStackTrace();
        }

        System.out.println("--------------------------");

        sql = "select * from person";
        try
        {
            List<Map<String, Object>> list = jdbcUtil.findMoreResult(sql, null);
            System.out.println(list);
        } catch (Exception e)
        {
            // TODO: handle exception
            e.printStackTrace();
        }

    }
```
		

 


 


后面改动Java的反射特性改写查找函数.

			