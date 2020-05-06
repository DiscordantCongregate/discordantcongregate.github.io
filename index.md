#!/usr/bin/php
<?php
define('LISTEN', true);
define('PID_FILE', './cboxbot.pid');

// Edit these lines for your Cbox and bot user. 
$box = array('srv' => 7, 'id' => 1, 'tag' => 'TrefoilAcademyForum');
$bot = array('name' => 'Dice', 'token' => 'AxrerT0zvn8RAtue', 'url' => '');
$msg = 'Hello world!';


$callmap = array(
	'/\bhello bot\b/iu' => 'bot_greet',
	'/\btime\?/iu' => 'bot_time',
	'/\bweather in ([a-zA-Z-0-9 ,]+)/iu' => 'bot_weather',
);

// Basic reply
function bot_time ($msg) {
	return date("M d Y H:i:s");
}

// An example incorporating message data. 
function bot_greet ($msg) {
	return "Hello ".$msg['name'];
}

// An example calling an external API
function bot_weather ($msg, $matches) {
	$place = $matches[1];

	if (!$place) {
		return;
	}

	$query = 'select * from weather.forecast where woeid in (select woeid from geo.places(1) where text="'.addslashes($place).'")';

	$url = 'https://query.yahooapis.com/v1/public/yql?q='.urlencode($query).'&format=json&u=c&env=store%3A%2F%2Fdatatables.org%2Falltableswithkeys';

	$out = file_get_contents($url);

	if (!$out) {
		return;
	}

	$wobj = json_decode($out);

	if (!$wobj->query->results) {
		return;
	}

	$cond = $wobj->query->results->channel->item->condition;
	$desc = $wobj->query->results->channel->description;

	return "[b]".$cond->text." ".$cond->temp."F [/b] (".$desc." at ".$cond->date.")";
}


// Do not edit past this point. 

set_time_limit(0);


$id = cbox_post($msg, $bot, $box, $error);

if (!$id) {
	echo $error;
}
else {
	echo "Posted ID $id\n";
}

if (!LISTEN) {
	exit;
}

// Synchronization.
// PID file is not removed on exit, but it is unlocked. A locked file indicates a running process.
$fp = fopen(PID_FILE, 'a+');

if (!flock($fp, LOCK_EX | LOCK_NB)) {
	echo "Could not lock PID file. Process already running?\n"; 
	exit;
}

ftruncate($fp, 0);
fwrite($fp, posix_getpid()."\n");
fflush($fp);

do {

	$msgs = cbox_get_msgs($id, $bot, $box);

	if (!$msgs || !is_array($msgs)) {
		sleep(5);
		continue;
	}

	$id = (int)$msgs[0]['id'];

	for ($i = 0; $i < count($msgs); $i++) {
	
		if ($msgs[0]['name'] == $bot['name']) {
			continue;	// Ignore bot's own messages.
		}
	
		$msgtext = $msgs[$i]['message'];
		
		foreach ($callmap as $expr => $func) {
			$matches = array();
			
			if (preg_match($expr, $msgtext, $matches)) {
			
				$reply = call_user_func($func, $msgs[$i], $matches);
				
				if ($reply) {
					cbox_post($reply, $bot, $box, $error);
				}
			}
		}
	}
	
	sleep(2);
} while (true);


function cbox_get_msgs ($id, $user, $box, &$error = '') {
	$srv = $box['srv'];
	$boxid = $box['id'];
	$boxtag = $box['tag'];
	
	$host = "www$srv.cbox.ws";
	$path = "/box/?boxid=$boxid&boxtag=$boxtag&sec=archive";
	$port = 80;
	$timeout = 30;

	$get = array(
		'i' => (int)$id,
		'k' => $user['token'],
		'fwd' => 1,
		'aj' => 1
	);	

	$req = '';
	$res = '';
	
	foreach ($get as $k => $v) {
		$path .= "&$k=".urlencode($v);
	}
	
	$hdr  = "GET $path HTTP/1.1\r\n";
	$hdr .= "Host: $host\r\n\r\n";
	
	$fp = fsockopen ($host, $port, $errno, $errstr, $timeout);

	if (!$fp) {
		$error = "Could not open socket: $errno - $errstr\n";
		return;
	}

	fputs ($fp, $hdr);
	
	while (!feof($fp)) {
		$res .= fgets ($fp, 1024);
	}
	
	fclose ($fp);

	if (!$res || !strpos($res, "200 OK")) {
		$error = "Bad response:\r\n $res";
		return;
	}

	$matches = array();

	preg_match_all('/\n([^\t\n]*)\t([^\t]*)\t([^\t]*)\t([^\t]*)\t'
.'([^\t]*)\t([^\t]*)\t([^\t]*)\t([^\t]*)\t/', $res, $matches);

	$msgs = array();

	$map = array('id', 'time', 'date', 'name', 'group', 'url', 'message');

	for ($m = 0; $m < count($map); $m++) {
		for ($i = 0; $i < count($matches[$m+1]); $i++) {
			$msgs[$i][$map[$m]] = $matches[$m+1][$i];
		}
	}

	return $msgs;
}


function cbox_post ($msg, $user, $box, &$error = '') {
	$srv = $box['srv'];
	$boxid = $box['id'];
	$boxtag = $box['tag'];
	
	$host = "www$srv.cbox.ws";
	$path = "/box/?boxid=$boxid&boxtag=$boxtag&sec=submit";
	$port = 80;
	$timeout = 30;

	$post = array(
		'nme' => $user['name'],
		'key' => $user['token'],
		'eml' => $user['url'],
		'pst' => $msg,
		'aj' => '1'
	);	

	$req = '';
	$res = '';
	
	foreach ($post as $k => $v) {
		$req .= "$k=".urlencode($v)."&";
	}
	$req = substr($req, 0, -1);
	
	$hdr  = "POST $path HTTP/1.1\r\n";
	$hdr .= "Host: $host\r\n";
	$hdr .= "Content-Type: application/x-www-form-urlencoded\r\n";
	$hdr .= "Content-Length: ".strlen($req)."\r\n\r\n";
	
	$fp = fsockopen ($host, $port, $errno, $errstr, $timeout);

	if (!$fp) {
		$error = "Could not open socket: $errno - $errstr\n";
		return;
	}

	fputs ($fp, $hdr.$req);
	
	while (!feof($fp)) {
		$res .= fgets ($fp, 1024);
	}
	
	fclose ($fp);

	if (!$res || !strpos($res, "200 OK")) {
		$error = "Bad response:\r\n $res";
		return;
	}

	$matches = array();

	preg_match('/1(.*)\t(.*)\t(.*)\t(.*)\t(.*)\t(.*)/', $res, $matches);

	$err = $matches[1];
	$id = $matches[6];

	if ($err) {
		$error = "Got error from Cbox: $err";
		return;
	}

	return $id;
}
?>
