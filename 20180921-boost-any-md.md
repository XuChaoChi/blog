---
title: 'Boost::any的使用以及用c++11标准库实现'
tags:
  - boost
keywords: 'any,boost,c++11'
category: boost
abbrlink: '20042906'
date: 2018-09-21 02:54:53
---

# Boost::any的使用以及用c++11标准库实现 #

## any ##
any是在boost中使用type_traits实现的一个类型存储转换工具类，可以方便的通过any存放数据并取出，any也被c++17收录

## any方法 ##
下面是any中可见的成员方法
### 构造方法 ###

	any()
	
	template<typename ValueType>
    any(const ValueType & value)

	any(const any & other)

	any(any&& other)//c++11支持

	template<typename ValueType>//c++11支持
        any(ValueType&& value
            , typename boost::disable_if<boost::is_same<any&, ValueType> >::type* = 0 // disable if value has type `any&`
            , typename boost::disable_if<boost::is_const<ValueType> >::type* = 0) // disable if value has type `const ValueType&&`
<!-- more -->
### 重载运算符 ###
	
	template<typename ValueType>
    any & operator=(const ValueType & rhs)

	any & operator=(any rhs)

	any & operator=(const any& rhs)//c++11

	any & operator=(any&& rhs)//c++11

	template <class ValueType>
        any & operator=(ValueType&& rhs)//c++11

### public成员函数 ###

	bool empty()
	
	void clear()

	const boost::typeindex::type_info& type() const	

	any & swap(any & rhs)

	inline void swap(any & lhs, any & rhs)

## any的使用 ##
- any提供了异常捕捉类，在转换类型不同时捕捉异常可以避免崩溃。
- any很好的解决了泛形模板确定类型后不能再传其他类型的问题，以前一般传基类，现在可以传any方便很多，比如vector之类的stl容器可以方便的存不同类型。

		#include "boost/any.hpp"
		#include "iostream"
		void printAny(boost::any & _testAny){
			//判断为空
			if(_testAny.empty()){
				std::cout << "empty" << std::endl;
				return;
			}
			//判断类型
			if(_testAny.type() == typeid(int)){
				int nValue = boost::any_cast<int>(_testAny);
				std::cout << nValue << std::endl;
			}
			else{
				//捕捉转换异常
				try{
					std::string strValue = boost::any_cast<std::string>(_testAny);
					std::cout << strValue << std::endl;
				}
				catch(const boost::bad_any_cast& _badCast){
					std::cout << "not string obj "<<_badCast.what() << std::endl;
				}
			}
		}
		
		
		int main(){
			int nTest = 10;
			boost::any anyTestInt(nTest);
			std::string strTest = "test";
			boost::any anyTestString(strTest);
			boost::any anyEmpty;
			boost::any anyDouble(1.2);
			printAny(anyTestInt);
			printAny(anyTestString);
			printAny(anyEmpty);
			printAny(anyDouble);
			std::cout << std::endl;	
			std::vector<boost::any>vecAny;
			vecAny.push_back(anyTestInt);
			vecAny.push_back(anyTestString);
			for(auto &vIter : vecAny){
				printAny(vIter);
			}
			getchar();
			return 0;
		}


## 用c++标准库实现any（只实现了c++11） ##

	#pragma once
	#include <typeinfo>
	#include <memory>
	
	class any{
	
	public:
		any():m_pContent(nullptr){}
	
		any(const any & other)
			: m_pContent(other.m_pContent ? other.m_pContent->clone() : nullptr){}
	
		//move
		any(any&& other)
			: m_pContent(other.m_pContent){
			other.m_pContent = nullptr;
		}
	
		//完美转发
		template <typename ValueType>
		any(ValueType && value,
			typename std::enable_if< !std::is_same<ValueType, any&>::value >::type* = nullptr,
			typename std::enable_if< !std::is_const<ValueType>::value >::type* = nullptr)
			: m_pContent(std::make_shared<holder<typename std::decay<ValueType>::type >>(std::forward<ValueType>(value))){
		}
	
		any &operator = (const any & rhs){
			any(rhs).swap(*this);
			return *this;
		}
	
		//move
		any & operator = (any && rhs){
			rhs.swap(*this);
			any().swap(rhs);
			return *this;
		}
	
		//完美转发
		template <typename ValueType>
		any & operator = (ValueType && rhs
			){
			any(std::forward<ValueType>(rhs)).swap(*this);
			return *this;
		}
	
		bool empty(){
			return m_pContent == nullptr;
		}
	
		void clear(){
			m_pContent = nullptr;
		}
	
		const std::type_info &type() const{
			return m_pContent ? m_pContent->type() : typeid(void);
		}
	
		any &swap(any &rhs){
			std::swap(rhs.m_pContent, rhs.m_pContent);
			return *this;
		}
	private:
		class placeholder{
		public:
			virtual ~placeholder(){}
	
			virtual const std::type_info  &type() = 0;
	
			virtual std::shared_ptr<placeholder> clone() const = 0;
		};
	
		template<typename ValueType>
		class holder: public placeholder{
		public:
			holder & operator = (const holder &) = delete;
	
			holder(const ValueType &value)
	 			:m_hold(value){}
	
			holder(ValueType &&value)
				:m_hold(std::forward<ValueType>(value)){}
	
			const std::type_info & type() override{
				return typeid(m_hold);
			}
	
			std::shared_ptr<placeholder> clone() const override{
				return std::make_shared<holder>(m_hold);
			}
	
			ValueType m_hold;
		};
		template<typename ValueType>
		friend ValueType* any_cast(any *);
		std::shared_ptr<placeholder> m_pContent = nullptr;
	};
	
	class bad_any_cast: public std::bad_cast{
	public:
		const char *what() const override{
			return "bad_any_cast: failed conversion using any_cast";
		}
	};
	
	template<typename ValueType>
	ValueType* any_cast(any *operand){
		return operand && operand->type() == typeid(ValueType)
			? &static_cast<any::holder<std::remove_cv< ValueType>::type>*>(operand->m_pContent.get())->m_hold 
			: nullptr;
	}
	
	template<typename ValueType>
	inline const ValueType *any_cast(const any *operand){
		return any_cast<ValueType>(const_cast<any*>(operand));
	}
	
	template<typename ValueType>
	ValueType any_cast(any &operand) {
		typedef typename std::remove_reference<ValueType>::type nonref;
		auto pResult = any_cast<nonref>(&operand);
		if (!pResult) {
			throw bad_any_cast();
		}
		typedef typename std::conditional<std::is_reference<ValueType>::value, ValueType
			, typename std::add_lvalue_reference<ValueType>::type>::type ref_type;
		return static_cast<ref_type>(*pResult);
	}
	
	template<typename ValueType>
	inline ValueType any_cast(const any &operand) {
		return any_cast(const_cast<any>(operand));
	}
	
	template<typename ValueType>
	inline ValueType any_cast(any&& operand){
		static_assert(
			std::is_rvalue_reference<ValueType&&>::value
			|| std::is_const<typename std::remove_reference<ValueType>::type>::value,
			"any_cast shall not be used for getting nonconst references to temporary objects"
			);
		return any_cast(std::forward<ValueType>(operand));
	}-
