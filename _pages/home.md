---
layout: homepage
permalink: /
author_profile: true
author: "Cisco IOS-XR Team"
sitemap: true
date: null

---

{% include base_path %}


<div class="feature__wrapper" style="margin-top: 5em;">
    <div class="feature__item--left">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="max-height: 2800px; max-width: 2000px;display: block; margin-left: auto; margin-right: auto;">
            <a href="{{ base_path }}/images/network_stack_layers.png"><img src="{{ base_path }}/images/network_stack_layers.png" alt="" /></a>
          </div>
          <div class="archive__item-body">
              <h2 class="archive__item-title">APIs at every Layer of the Stack</h2>
              <div class="archive__item-excerpt" style="font-size: 0.65em;">
                <p>Model Driven APIs (YANG, protobuf) at every layer of the IOS-XR stack are exposed over suitable RPC mechanisms such as gRPC and Netconf. Contractually Versioned, easily transformed through tools such as ydk and protoc to bindings in the language of your choice. Automate the state of the network device or take control of your control-plane's state machine, all while monitoring modeled data through a telemetry stream </p>
              </div>
              <a href="https://xrdocs.io/programmability/"><button type="button" class="btn btn-primary btn-small">Model driven Manageability</button></a>
              <a href="https://xrdocs.io/cisco-service-layer/"><button type="button" class="btn btn-primary btn-small">Service Layer APIs</button></a>
              <a href="https://xrdocs.io/telemetry/"><button type="button" class="btn btn-primary btn-small">Telemetry</button></a>
          </div>
      </div>
    </div>
</div>


<div class="feature__wrapper">    
<div class="feature__item--left" style="margin-bottom: 2em;">
      <div class="archive__item" style="margin-left: 2em;">
          <div class="archive__item-teaser center" style="max-height: 400px; max-width: 400px;display: block;
           margin-left: auto; margin-right: auto;">
            <img src="{{ base_path }}/images/mdp_logos.png" alt="" />
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title">YANG based APIs</h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
            <p> Corresponds to the Manageability layer, allowing the user to programmatically influence the state of the IOS-XR stack. The APIs are modeled using YANG and represent nodes of an internal database called SYSDB. The APIs can be utilized through different techniques: </p>
            <ul>
                <li><b>YDK:</b>Utilize YDK-gen to create bindings in python,c++ or golang and send/receive YANG data in XML format through providers such as netconf</li>
               <li><b>ncclient (YANG-XML):</b>Utilize ncclient to setup netconf sessions with the device and send/receive YANG data encoded in XML</li>
               <li><b>gRPC (YANG-XML):</b>Utilize gRPC, create clientes in language of  choice and send/receive YANG data in a JSON formatC</li>
            </ul>   
            </div>
            <a href="http://ydk.io"><button type="button" class="btn btn-primary btn-small">YDK</button></a>
            <a href="https://github.com/ncclient/ncclient"><button type="button" class="btn btn-primary btn-small">ncclient</button></a>
            <a href="https://xrdocs.io/programmability/tutorials/2016-11-03-grpc-in-python-for-ios-xr/"><button type="button" class="btn btn-primary btn-small">Yang over gRPC</button></a>        
            </div>
      </div>
</div>
</div>




<div class="feature__wrapper">
    <div class="feature__item--right" style="margin-bottom: 2em;">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="max-height: 400px; max-width: 400px;display: block; margin-left: auto; margin-right: auto;">
            <a href="{{ base_path }}/images/slapi-logos.png"><img src="{{ base_path }}/images/slapi-logos.png" alt="" /></a>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title">Service Layer APIs</h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p>APIs at the network infrastructure layer of the IOS-XR stack. Enables model-driven access over gRPC to functionality verticals such as RIB , Label Switch Database, Interface Events and BFD events with more functionalities coming in the future. </p>
              <ul>
                  <li><b>Performance:</b> Highly Performant API enabling route programming at a rate of 20000-30000 routes/second</li>
                  <li><b>Model Driven:</b> Protobuf IDL based model for RPC and data structure definitions</li>
                 <li><b>Multi-Language Client support:</b>Choose any language binding supported by gRPC to write a service-layer API client</li>
                 </ul>   
              </div>
            <a href="https://xrdocs.io/cisco-service-layer/"><button type="button" class="btn btn-primary btn-small">Service Layer APIs</button></a>
        </div>
      </div>
    </div>
</div>




<div class="feature__wrapper">    
<div class="feature__item--left" style="margin-bottom: 2em;">
      <div class="archive__item" style="margin-left: 2em;">
          <div class="archive__item-teaser center" style="margin-top: 2em;max-height: 800px; max-width: 800px;display: block;
           margin-left: auto; margin-right: auto;">
            <img src="{{ base_path }}/images/telemetry.png" alt="" />
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title">Streaming Telemetry</h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
            <p>Streaming Telemetry helps deliver a push-based technique for getting operational data out of an IOS-XR box - using model-driven YANG paths with the capacity to integrate with a wide variety of open source tools like pipeline, kafka, influxdb, ELK, prometheus and more or just write a gRPC telemetry client of your own and subscribe to the stream </p>
            <ul>
                <li><b>Model Driven:</b>Structured data is sent back to collectors/clients allowing easy parsing and integration with larger remediation/alarm systems.</li>
               <li><b>Highly Scalable and Performant:</b>Telemetry has been shown to beat SNMP both at scale and at the rate at which data can be streamed out without negligible effect on the resources of the network device.</li>
               <li><b>Wide variety of integrations:</b> Open Source tools and integrations available for a vast set of monitoring tools in the industry.</li>
            </ul>   
            </div>
            <a href="https://xrdocs.io/telemetry"><button type="button" class="btn btn-primary btn-small">Streaming Telemetry</button></a>        
            </div>
      </div>
</div>
</div>


<div class="feature__wrapper">
    <div class="feature__item--right" style="margin-bottom: 2em;">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="max-height: 400px; max-width: 400px;display: block; margin-left: auto; margin-right: auto;">
            <a href="{{ base_path }}/images/ztp_cli_hooks.png"><img src="{{ base_path }}/images/ztp_cli_hooks.png" alt="" /></a>
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title">Still in love with CLI? - Automate using bash and python libraries!</h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p>As part of the Zero-touch provisioning infrastructure, IOS-XR provides the option to automate CLI operations in either bash or python in the IOS-XR shell environment. This enables a large variety of integrations - including native scripts and offbox integrations with tools like Ansible</p>
              <ul>
                  <li><b>Complete CLI support:</b> Automate CLI operations such as "show commands", "config merge", "config replace"</li>
                  <li><b>Available in bash and python:</b> Choose the scripting language you prefer and integrate with ztp, cronjobs, onbox-apps and more </li>
                  </ul>
            <a href="https://xrdocs.io/software-management/tutorials/2016-08-26-working-with-ztp/#ztp_helpersh"><button type="button" class="btn btn-primary btn-lg">IOS-XR Bash ZTP hooks</button></a>       
            <a href="https://github.com/ios-xr/iosxr-ztp-python"><button type="button" class="btn btn-primary btn-lg">IOS-XR Python ZTP Library</button></a>
        </div>
      </div>
    </div>
</div>
