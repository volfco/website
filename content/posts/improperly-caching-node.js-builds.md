+++
date = 2021-07-29T22:25:38Z
tags = ["node.js", " github-actions", " cicd"]
title = "Improperly caching node.js builds"

+++
Quick guide on how to improperly cache node.js builds. The logic here in generic, but I'm doing this on Github Actions. It should work everywhere else, but be warned. 

## Overview

What am I trying to do here? Well, make building of a `react-static` application as fast as possible for my CI/CD pipeline. I can't really tell you much more than it's a react static application, because I'm clueless on node.js and was asked to do this project without knowing much about the ecosystem. 

But, whatever. The core components are react, webpack, and bazel. There's also some custom magic that is used to load content in during the build process. I'll cover improving that in another post. 

The app is built against node 10.23.1. I know. I know. Not my pig.

## node tweaks

I'm not sure of how effective these tweaks are on build time. Due to the variable performance of GitHub's Action environment, these only show a marginal improvement in performance between two runs. 

    env:  
        NODE_OPTIONS: "--max-old-space-size=4096 --v8-pool-size=16"  
        UV_THREADPOOL_SIZE: 16

Those above environmental variables are present for both `yarn install` and `yarn build`. Setting `NODE_ENV` to `production` during _yarn install_ broke a few dependencies, resulting in the build failing- so it was removed. Not really worth investigating why.

## proper caching

What I found is that all the following files need to be cached in order to avoid resolving packages. 

    ~/.npm
    ~/.cache/yarn/v6
    
    ./yarn.lock
    ./node_modules/*
    ./node_modules/.*

usually, I see node_modules as the directory to cache. While this is needed, yarn will still crawl over the tree and verify each thing is correct. **yarn.lock** is what's used to avoid resolving packages, and looks like it does a much quicker verification of existing contents. 

This is safe to do because the node version we're using is fixed. If the node version changes, you will need to re-build everything. 

Both \~/.npm and \~/.cache/yarn/v6 were also populated after a build, so I'm including that in my build cache. 

## conclusion

The resulting github action looks like this:

    - name: yarn cache
      uses: actions/cache@v2
      with:
        path: |     
          ~/.npm      
          ~/.cache/yarn/v6      
          ./yarn.lock      
          ./node_modules/*      
          ./node_modules/.*      
         key: build-node-10-npm-assets-${{ hashFiles('yarn.lock') }}
         
    - name: yarn install
      run: |    
        yarn install --no-progress --non-interactive
        yarn client install --no-progress --non-interactive --prefer-offline  
      env:    
        NODE_OPTIONS: "--max-old-space-size=4096 --v8-pool-size=16"    
        UV_THREADPOOL_SIZE: 16
        
    - name: yarn build  
      run: |    
        yarn store-client build --no-color --emoji false --non-interactive  
      env:    
        NODE_ENV: production    
        NODE_OPTIONS: "--max-old-space-size=4096 --v8-pool-size=16"
        UV_THREADPOOL_SIZE: 16

Before these improvements, the average build time was around 10 minutes. With these improvements, the average time dropped to 5 minutes.