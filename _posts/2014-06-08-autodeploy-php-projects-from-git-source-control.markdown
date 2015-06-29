---
layout: post
title: "Autodeploy php projects from Git source control"
date: 2014-06-08 12:51:02 +1000
comments: true
author: Nam Nguyen
categories: [PHP]
---

Bitbucket is the great private Git repository free for team less than 5 users, with a bit of configuration we can
hook Bitbucket to automatic deploy git branches to a live server.

<!-- more -->

Follow this article to setup SSH access and config Bitbucket for your project

Few important things to remember:

* You have to run every setup as web user, for example www-data or apache or any user you use to run php script
* The user should have permission to access git clone project
* Restart web server after making changes

[http://jonathannicol.com/blog/2013/11/19/automated-git-deployments-from-bitbucket/]("http://jonathannicol.com/blog/2013/11/19/automated-git-deployments-from-bitbucket/#comment-629424" "Automated")

I have changed the script to make it more flexible and secured:

* Add .htaccess to limit access to log files and main command file
* Add support for different type of deployment (i.e test/production environment)
* Add support for PHP versions older than 5.2, which don't have json_decode function
* Run deployment process in different php process to free Bitbucket from waiting for the deployment to be completed


``` php .bitbucket-hook-prod.php
<?php
if(isset($_POST['payload'])) {
    // save payload to file
    file_put_contents(getcwd().'/bitbucket.log', serialize($_POST['payload']));
    // call command
    exec('nohup php gitdeploy.php > /dev/null 2>&1 &');
}
```

Main deployment script

``` php .gitdeploy.php
<?php
// increase execution time
ini_set('max_execution_time', 1800);
define('REPO_DIR', '/path/to/yourproject.git');
define('PROD_WEB_ROOT_DIR', '/path/to/public_html');
define('TEST_WEB_ROOT_DIR', '/path/to/public_html_test');
define('GIT_BIN_PATH', 'git');
define('GIT_DEPLOY_PATH', getcwd());
define('PROD_BRANCH_NAME', 'production');
define('TEST_BRANCH_NAME', 'test');

// get payload from latest
$payload = unserialize(file_get_contents(GIT_DEPLOY_PATH . '/bitbucket.log'));
$payload = (function_exists('json_decode')) ? json_decode(str_replace("\\", "", $payload)) : fromJSON(
    str_replace("\\", "", $payload)
);

// check for committed test and prod
$updateProd = false;
$updateTest = false;
if (!empty($payload->commits)) {
    foreach ($payload->commits as $commit) {
        $branch = $commit->branch;
        if ($branch === PROD_BRANCH_NAME || isset($commit->branches) && in_array(PROD_BRANCH_NAME, $commit->branches)) {
            $updateProd = true;
        }
        if ($branch === TEST_BRANCH_NAME || isset($commit->branches) && in_array(TEST_BRANCH_NAME, $commit->branches)) {
            $updateTest = true;
        }
    }
}

if ($updateProd) {
    deploy(PROD_BRANCH_NAME, PROD_WEB_ROOT_DIR);
    // additional modifications (change config file, run sql script, etc)
    // send email notifications to team
}
if ($updateTest) {
    deploy(TEST_BRANCH_NAME, TEST_WEB_ROOT_DIR);
    // additional modifications (change config file, run sql script, etc)
    // send email notifications to team
}


function deploy($branch, $webdir)
{
    $output = "";
    exec('cd ' . REPO_DIR . ' && ' . GIT_BIN_PATH . ' fetch 2>&1', $output);
    // checkout production branch
    exec(
        'cd ' . REPO_DIR . ' && GIT_WORK_TREE=' . $webdir . ' ' . GIT_BIN_PATH . ' checkout -f ' . $branch . ' 2>&1',
        $output
    );
    // get commit hash from current production branch
    $commit_hash = shell_exec('cd ' . REPO_DIR . ' && ' . GIT_BIN_PATH . ' rev-parse --short HEAD');
    // log the deployment
    file_put_contents(
        GIT_DEPLOY_PATH . '/deploy.log',
        date('m/d/Y h:i:s a') . " Deployed branch: " . $branch . " Commit:" . $commit_hash . "\n",
        FILE_APPEND
    );
}


/**
 * https://code.google.com/p/simplejson-php/
 * Parses a JSON string into a PHP variable.
 * @param string $json The JSON string to be parsed.
 * @param bool $assoc Optional flag to force all objects into associative arrays.
 * @return mixed Parsed structure as object or array, or null on parser failure.
 */
function fromJSON($json, $assoc = false)
{

    /* by default we don't tolerate ' as string delimiters
       if you need this, then simply change the comments on
       the following lines: */

    // $matchString = '/(".*?(?<!\\\\)"|\'.*?(?<!\\\\)\')/';
    $matchString = '/".*?(?<!\\\\)"/';

    // safety / validity test
    $t = preg_replace($matchString, '', $json);
    $t = preg_replace('/[,:{}\[\]0-9.\-+Eaeflnr-u \n\r\t]/', '', $t);
    if ($t != '') {
        return null;
    }

    // build to/from hashes for all strings in the structure
    $s2m = array();
    $m2s = array();
    preg_match_all($matchString, $json, $m);
    foreach ($m[0] as $s) {
        $hash = '"' . md5($s) . '"';
        $s2m[$s] = $hash;
        $m2s[$hash] = str_replace('$', '\$', $s); // prevent $ magic
    }

    // hide the strings
    $json = strtr($json, $s2m);

    // convert JS notation to PHP notation
    $a = ($assoc) ? '' : '(object) ';
    $json = strtr(
        $json,
        array(
            ':' => '=>',
            '[' => 'array(',
            '{' => "{$a}array(",
            ']' => ')',
            '}' => ')'
        )
    );

    // remove leading zeros to prevent incorrect type casting
    $json = preg_replace('~([\s\(,>])(-?)0~', '$1$2', $json);

    // return the strings
    $json = strtr($json, $m2s);

    /* "eval" string and return results.
       As there is no try statement in PHP4, the trick here
       is to suppress any parser errors while a function is
       built and then run the function if it got made. */
    $f = @create_function('', "return {$json};");
    $r = ($f) ? $f() : null;

    // free mem (shouldn't really be needed, but it's polite)
    unset($s2m);
    unset($m2s);
    unset($f);

    return $r;
}

```

To prevent public view of log file

``` php .htaccess file
<Files "*.log">
Order Allow,Deny
Deny from all
</Files>

<Files "gitdeploy.php">
Order Allow,Deny
Deny from all
</Files>
```

Enjoy