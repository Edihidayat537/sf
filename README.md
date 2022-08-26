2018年8月之后，shopee开始使用新接口，需要进行授权操作

1.授权

public function getAuth(){
    /**
     * @param ShopApiShopee $model
     */
    $httpType = ((isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] == 'on') || (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https'))
        ? 'https://' : 'http://';
    $redirectUrl = $httpType . $_SERVER['HTTP_HOST'] . '/';
    $token = hash('sha256', $this->_ShopApiShopee->shopee_key . $redirectUrl);
    $url = self::SHOPEE_GET_AUTH_URL
        . '?id=' . $this->_ShopApiShopee->shopee_shop_id
        . '&token=' . $token
        . '&redirect=' . $redirectUrl;
    header("Location: " . $url);
    $this->_ShopApiShopee->token = $token;
    if (!$this->_ShopApiShopee->save()) {
        return self::fail(self::CODE_SAVE_ERROR,serialize($this->_ShopApiShopee->getErrors()));
    }
}

2.拼接获取订单信息

static function getOrderList($shopApiShopee, $days = 1, $offsetDay = 0)
{
    if(!empty($shopApiShopee) && $shopApiShopee instanceof ShopApiShopee )
    {
        $to_time_stamp = time() - $offsetDay * 86400;
        $from_time_stamp = $to_time_stamp - 86400 * $days;
        $arr = array(
            'partner_id' => (int)$shopApiShopee->shopee_partner_id,
            'shopid' => (int)$shopApiShopee->shopee_shop_id,
            'timestamp' => time(),
            'create_time_from'          => $from_time_stamp,
            'create_time_to'            => $to_time_stamp,
            'pagination_offset'             => 0,
            'pagination_entries_per_page'  => 100
        );
        $result = self::_getCurlResponse($shopApiShopee, self::SHOPEE_GET_ORDER_LIST_URL, $arr);
        return $result;
    }
    else
    {
        return self::fail(self::CODE_NO_FIND, 'shopee param invalid:' . serialize($shopApiShopee));
    }
}

private static function _getCurlResponse($shopApiShopee, $url, $arr){
    $arr = json_encode($arr);
    $contentLength = strlen($arr);
    $strConcat = $url.'|'.$arr;
    $authorizationKey = hash_hmac('sha256', $strConcat, $shopApiShopee->shopee_key);
    $header = array(
        'Content-Type:application/json',
        "Content-Length:" . $contentLength,
        'Authorization:' . $authorizationKey
    );
    $params = array(
        'base_uri'   => $url,
        'headers'    => $header,
        'verify'     => false,
        'body'    => $arr
    );
    return self::_curl($params);
}

3.使用curl请求orderList数据

    private static function _curl($params){
        $ch = curl_init($params['base_uri']);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, FALSE);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, FALSE);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $params['headers']);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
//        curl_setopt($ch, CURLOPT_HEADER, TRUE); 返回打印信息中包含头文件
        curl_setopt($ch, CURLOPT_POST, TRUE);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $params['body']);
        $str = curl_exec($ch);
        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        if( $status === self::CODE_SUCCESS ){
            return self::success(json_decode($str,true), self::CODE_SUCCESS);
        }else{
            return self::fail(json_decode($str,true), $status);
        }
    }
