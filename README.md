qserouter
=========

only 2k的可配置的URL 路由实现 qserouter

具体代码如下

  <?php
	/**
	 * 简单的URL 路由解析器
	 * 
	 * 匹配规则 区分大小写
	 * 'rule' => '/microblog/{operate}/{action}/*',
	 * 
	 * @author vb2005xu.iteye.com
	 */
	class Router {
		
		private $routes = array();
		
		// url 模型参数,用于标识资源对象
		const module = 'mod';
		const operate = 'op';
		const action = 'act';
		
		const default_module = 'default';
		const default_operate = 'application';
		const default_action = 'index';
		
		// 不支持url重写的情况下读取 qurlid 这个参数后的串作为pathinfo 
		const qurlid = 'q';
		
		function __construct(array $routes){		
			
			if (!empty($routes)){
				foreach ($routes as $name=>$route){
					if (isset($route['rule']) && isset($route['default']))
						$this->routes[$name] = $route ;
				}
			}
			if (!isset($this->routes['_default_'])){
				$this->routes = array_merge(array(
					'_default_' => array(
						'rule' => '/{operate}/{action}/*',
						'default' => array(
							'udi' => self::normalizeUDI('application','index')
						)		
					)
				),$this->routes);
			}
		}
		
		function mapping($name,array $route = null){
			if ($route){
				if (isset($route['rule']) && isset($route['default'])){
					$this->routes[$name] = $route ;
				}
			}
		}
		
		// $pathinfo,$rule 末尾均不能带 / 符号
		private function match($pathinfo,$rule){
			//比对 $pathinfo 和 $route['pattern']
			// /books/13/csx <==> /books/{id}/{keyname}
			// 定位模式中 第一个 路由变量[{var}] 出现的位置,并比较
			$pos = strpos($rule,'{') ;
			if ($pos)			
				return strcmp(substr($rule, 0,$pos-1),substr($pathinfo, 0,$pos-1)) == 0 ;
			// 模式中可能 不包含 路由变量[{var}]
			return strlen($pathinfo)>strlen($rule) ?
				strcmp(substr($pathinfo,0,strlen($rule)),$rule)==0 : false ;
		}
		
		// pathinfo 按 规则解析成 url_params 数组
		private function pathinfo_to_array($pathinfo,$rule){
					
			$path_parts = explode('/', substr($pathinfo,1));
			$rule_parts = explode('/', substr($rule,1));
			$result = array();
					
			while(!empty($rule_parts)){
				
				$r = array_shift($rule_parts);
				if ($r == '*' || $r == '?') continue;
				$p = array_shift($path_parts);
				if ($p === NULL) continue ;
				if (strpos($r,'{') === false) continue ;
				
				// 当前是路由变量[{var}]
				$var = substr($r,1,strlen($r)-2);
				$result[strtolower($var)] = $p ;
			}
	//		dump($path_parts);dump($rule_parts);dump($result);die;
			// 解析额外参数,路径部分除指定的路由部分外,其它都使用 /key/value 
			while(!empty($path_parts)){					
				$k = array_shift($path_parts);$v = array_shift($path_parts);
				if (empty($k)) continue ;
				$result[strtolower($k)] = $v ;					
			}		
			return $result ;
		}
		
		// 填充默认参数
		private function __fill_default_params(&$url_params,array $default_params){
			if (!$default_params) return;
			
			$udi = isset($default_params['udi']) && is_array($default_params['udi']) ? $default_params['udi'] : NULL;
			$params = isset($default_params['params']) && $default_params['udi'] ? $default_params['params'] : NULL;
			
			if ($udi){
				// 替换路由定义文件中的 资源标识id
				$replace_pairs = array('module'=>self::module,'operate'=>self::operate,'action'=>self::action);
				foreach ($replace_pairs as $k=>$v){
					if (isset($url_params[$k])){
						// 存在就进行 数组键 替换
						$url_params[$v] = empty($url_params[$k]) ? $udi[$v] : $url_params[$k];
						unset($url_params[$k]);
					}
					else {
						$url_params[$v] = $udi[$v] ;
					}
				}
			}
			
			if ($params){
				$url_params = array_merge($params,$url_params);
			}
		}
		
		protected function parser($pathinfo,$route_name=null){ 
			// $routes
			if ($route_name){
				if (!isset($this->routes[$route_name])) return null ;
				$routes = array($route_name=>$this->routes[$route_name]) ;
			}else
				$routes = $this->routes ;
			
			$pathinfo = trim($pathinfo);
			// /books/show => /books/show/ 不同		
			//  移除末尾的 / 字符		
			if (preg_match('/\\/$/',$pathinfo))
				$pathinfo = preg_replace('/\\/$/','',$pathinfo);
				
			foreach (array_reverse($routes) as $name =>$route){
				//  移除末尾的 / 字符
				$route['rule'] = trim($route['rule']);
				if (preg_match('/\\/$/',$route['rule']))
					$route['rule'] = preg_replace('/\\/$/','',$route['rule']);
				
				if ($this->match( $pathinfo,$route['rule'] )){
					$url_params = $this->pathinfo_to_array($pathinfo,$route['rule']);
					// 合并 $url_params , $route['default'] 成 $_GET数组
					
					$this->__fill_default_params($url_params,$route['default']);
					
					return $url_params ;
				}
			}
			return null ;
		}
		
		// 过滤 HTTP GET 请求数组
		function filter(& $request,$pathinfo){
			
			if (empty($pathinfo)){
				$url_params = $this->parser($pathinfo,'_default_');			
			}
			
			if (!isset($url_params)) // 是pathinfo格式
				$url_params = $this->parser($pathinfo); 
			
			// 改变原来的请求数组
			if (!empty($url_params))
				$request = array_merge($request,$url_params);
			
			return $request;
		}
		
		/**
		 * 返回 pathinfo 字符串
		 * @return string
		 */
		static function pathinfo(){
			
			static $once = false;
			static $pathinfo = null;
			if ($once) return $pathinfo;
			
			$once = true;

			$pathinfo = !empty($_SERVER['PATH_INFO']) ? $_SERVER['PATH_INFO'] :
				(empty($_SERVER['ORIG_PATH_INFO']) ? '' : $_SERVER['ORIG_PATH_INFO']);
			
			if (!empty($pathinfo)) return $pathinfo;
			
			// No PATH_INFO?... What about QUERY_STRING?
			$pathinfo = (isset($_SERVER['QUERY_STRING'])) ? $_SERVER['QUERY_STRING'] : @getenv('QUERY_STRING');
			if (empty($pathinfo)){
				$pathinfo = null;
				return $pathinfo;
			}
			
			if (!isset($_GET[self::qurlid]) || empty($_GET[self::qurlid])) {
				$pathinfo = null;
				return $pathinfo;
			}
		
			$pathinfo = $_GET[self::qurlid];
			return $pathinfo;
		}
		
		static function normalizeUDI($operate=self::default_operate,$action=self::default_action,$module=self::default_module){
			$udi = array(
				self::module => empty($module) ? self::default_module : $module,
				self::operate => empty($operate) ? self::default_operate : $operate,
				self::action => empty($action) ? self::default_action : $action,
			);
			$udi = array_change_key_case($udi, CASE_LOWER);
			return $udi;
		}
		
		static function formatUDI(array $udi=null){
			
			$udi = $udi ? array_change_key_case($udi, CASE_LOWER) : array();
			
			if (!isset($udi[self::module]))
				$udi[self::module] = self::default_module;
			
			if (!isset($udi[self::operate]))
				$udi[self::operate] = self::default_operate;
				
			if (!isset($udi[self::action]))
				$udi[self::action] = self::default_action;
				
			return $udi;
		}
		
		static function toUDIString(array $udi)
		{
			$udi = self::formatUDI($udi);
			return "{$udi[self::operate]}/{$udi[self::action]}@{$udi[self::module]}";
		}
	}


  <?php
	// 路由配置数组,最下面的最先解析

	return array(	
		/*
		 * 默认路由
		 * 当没有任何匹配的路由规则时,使用默认的路由规则
		 */
		'_default_' => array(
			'rule' => "/{operate}/{action}/*" ,
			'default' => array(
				'udi' => Router::normalizeUDI('application','index')
			)
		) ,
		
		'books' => array(
			'rule' => '/books/{id}/{keyname}' ,
			'default' => array(
				'udi'=> Router::normalizeUDI('book','view'),
			) ,
		) ,
		
		'books/show' => array(
			'rule' => '/books/show/{id}/{uid}' ,
			'default' => array(
				'udi'=> Router::normalizeUDI('book','show'),
				'params'=>array('id'=>13)
			) ,
		) ,
		
		'_pqadmin_' => array(
			'rule' => '/admin/{operate}/{action}/*',
			'default' => array(
				'udi'=> Router::normalizeUDI('application','index','admin'),
			) ,
		) ,
	);

  <?php
	require 'router.php';

	function dump($vars, $label = '', $return = false)
	{
		if (ini_get('html_errors')) {
			$content = "<pre>\n";
			if ($label != '') {
				$content .= "<strong>{$label} :</strong>\n";
			}
			$content .= htmlspecialchars(print_r($vars, true));
			$content .= "\n</pre>\n";
		} else {
			$content = $label . " :\n" . print_r($vars, true);
		}
		if ($return) { return $content; }
		echo $content;
		return null;
	}

	$routes = new Router(require('mapping.php'));

	$testarr = array(
		'/',
		'/abc',
		'/abc/d',
		'/books/13',
		'/books/13/java',
		'/admin',
		'/admin/',
		'/admin/abc',
		'/admin/abc/d',
	);

	foreach ($testarr as $item){
		dump($routes->filter($_GET,$item),$item);	
	}



	?>



