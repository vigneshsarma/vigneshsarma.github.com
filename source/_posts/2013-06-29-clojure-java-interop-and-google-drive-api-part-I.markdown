---
layout: post
title: "Clojure Java Inter-op and Google Drive Api - Part I"
date: 2013-06-28 01:42
comments: true
categories: clojure google drive java
---
One of the biggest advantages of Clojure claimed by its community is its interoperability with java. I decided to take take them on there word and test it out with Google drive api, and see how flexible and native it feels.

We will convert a simple program given on the main documentation page of Google Drive api page to a Clojure program. First we will try to keep it as same as there implementation having least amount of changes. Later we will try to make it more Clojure like, by using its native libraries and using Google libraries to a minimum.

You will have to do the basic setup form this [page](https://developers.google.com/drive/quickstart-java). May be even give a quick look a the java implementation, because we will be following it quite faithfully.

##Setup
First you should create a new lein project:
```clojure
$ lein new sample-drive app
```
This will create a set of files and some basic stuff. Now you will have to add some dependencies add required libraries to the `project.clj` file.
You should be able to find this file at the root of the new project `lein` created for you. Change it to look some thing like this.
```clojure
(defproject simple-drive "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.5.1"]
                 [com.google.oauth-client/google-oauth-client "1.15.0-rc"]
                 [com.google.http-client/google-http-client-jackson "1.15.0-rc"]
                 [com.google.apis/google-api-services-drive "v2-rev79-1.15.0-rc"]]
  :main simple-drive.core)
```
`google-api-services-drive` is the the only library explicitly mentioned on the download page. But the other (`google-oauth-client`, `google-http-client-jackson`) are dependencies. You could use third party libraries that provide similar functionality like, `clj-oauth2`.

Another interesting not is we are getting them from maven repositories, which are essentially cpan or pypy like function for java. And the fact that we are even able to leverage that from Clojure is amazing.

##Program
The program itself is a single function in the given sample, but will split it up slightly so that we can keep the main program in a single let statement. Much of the code is java directly converted to clojure.
```clojure
(ns simple-drive.core
  (:gen-class))
(import '(com.google.api.client.googleapis.auth.oauth2
          GoogleAuthorizationCodeFlow GoogleCredential
          GoogleTokenResponse GoogleAuthorizationCodeFlow$Builder)
        '(com.google.api.client.http FileContent HttpTransport)
        '(com.google.api.client.http.javanet NetHttpTransport)
        '(com.google.api.client.json JsonFactory)
        '(com.google.api.client.json.jackson JacksonFactory)
        '(com.google.api.services.drive Drive DriveScopes Drive$Builder)
        '(com.google.api.services.drive.model File))
(require 'clojure.java.io)

;; you have to get your own from google as mentioned in the help page.
(def CLIENT_ID "SECRET_ID")
(def CLIENT_SECRET "SECRET_KEY")

(def REDIRECT_URI "urn:ietf:wg:oauth:2.0:oob")

(defn get-auth-code [url]
  (println "Please open the following URL in your browser "
           "then type the authorization code:")
  (println (str "  " url))
  (read-line))

(defn insert-file []
  (let [body (new File)]
    (doto body
      (.setTitle "Readme")
      (.setDescription "A test document")
      (.setMimeType "text/plain"))
    body))

(defn do-the-stuff
  []
  (let [httpTransport (new NetHttpTransport)
        jsonFactory (new JacksonFactory)
        flow (-> (new GoogleAuthorizationCodeFlow$Builder httpTransport
                      jsonFactory CLIENT_ID CLIENT_SECRET
                      (list (. DriveScopes DRIVE)))
                 (.setAccessType "online")
                 (.setApprovalPrompt "auto")
                 (.build))
        url (-> (.newAuthorizationUrl flow)
                (.setRedirectUri REDIRECT_URI)
                (.build))
        code (get-auth-code url)
        response (-> (.newTokenRequest flow code)
                     (.setRedirectUri REDIRECT_URI)
                     (.execute))
        credential (-> (new GoogleCredential)
                       (.setFromTokenResponse response))
        ;; Create a new authorized API client
        service (-> (new Drive$Builder httpTransport
                         jsonFactory credential) (.build))
        body (insert-file)
        fileContent (clojure.java.io/file "README.md")
        mediaContent (new FileContent "text/plain" fileContent)
        file (-> (.files service)
                 (.insert body mediaContent)
                 (.execute))]
    (println "File ID:" (.getId file))))

(defn -main
  "This is the equvalent of public static void main."
  [& args]
  ;; work around dangerous default behaviour in Clojure
  (alter-var-root #'*read-eval* (constantly false))
  (do-the-stuff))
```

Some of the stuff you might note, are that we have to explicitly import all the java inner classes. In java these are available to you if you just import the containing class.
The whole program is written in a imperative manner, you set a whole bunch of variables, in between you do a few things.
Instead of `(new classname &args)` we could have just said `(classname. &args)`.
